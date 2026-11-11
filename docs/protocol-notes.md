# ELAN 04f3:0c77: protocol and firmware notes

Findings from bringing the reader up on an ASUS ExpertBook B9403CVAR in
November 2026. Nothing here comes from a capture of the Windows driver. Every
command sent to the sensor was taken from an existing open-source driver
(`elanmoc`, `elanmoc2`) and tried on the hardware.

## The device

| | |
|---|---|
| USB ID | `04f3:0c77`, product string `ELAN:ARM-M4` |
| `bcdDevice` | 3.16; the firmware version query returns 0x316 |
| Sensor size | 51 × 149 (reply to `00 0c`) |
| Interface | One vendor-specific interface, no kernel driver |
| Endpoints | Bulk OUT 0x01–0x04, bulk IN 0x81–0x84, 64-byte packets |

Commands go out on 0x01. Replies to commands that wait for a finger come in on
0x84, all other replies on 0x83.

## Which driver

| Driver | Result on 04f3:0c77 |
|---|---|
| `elan` (image sensors) | Does not work. Reported in upstream issues #447 and #510: the capture command is never answered. Not re-tested here. |
| `elanmoc` (upstream) | Opens the device: status, set mode, firmware version, sensor size and enrolled count are all answered. Enrollment fails at once: the re-enroll check `40 ff 22` is never answered. |
| `elanmoc2` (Davide Depau) | Works, with the changes below. |

## Commands used

All start with `0x40`. Lengths are the full packet sizes.

| Command | Bytes | Out | In | Reply endpoint | Tested |
|---|---|---|---|---|---|
| Enrolled count | `40 ff 04` | 3 | 2 | 0x83 | Yes |
| Identify (waits for a finger) | `40 ff 03` | 3 | 2 | 0x84 | Yes |
| Finger info | `40 ff 12 <slot>` | 4 | 64 | 0x83 | Yes |
| Enroll sample (waits for a finger) | `40 ff 01 <slot> <total> <done> 00` | 7 | 2 | 0x84 | Yes |
| Collision check | `40 ff 10` | 3 | 3 | 0x83 | Yes |
| Commit | `40 ff 11 <f0 \| slot+5> <user id>` | 72 | 2 | 0x83 | Yes |
| Delete | `40 ff 13 <f0 \| slot+5> <user id>` | 72 | 2 | 0x83 | Yes |
| Wipe all | `40 ff 99` | 3 | 0 | – | Yes |
| Abort | `40 ff 02` | 3 | 2 | 0x83 | Sent on cancellation; reply not read |
| Status | `40 ff 00` | 3 | 2 | 0x83 | Only in the experimental patch |

Second reply byte of the finger commands:

| Value | Meaning |
|---|---|
| `0x00`–`0x0f` | Success; for identify, the slot that matched |
| `0x41`–`0x44` | Move the finger down / right / up / left |
| `0xfb` | Sensor dirty or wet |
| `0xfd` | Finger read but not recognised |
| `0xfe` | Too little finger area; the touch does not count |
| `0xdd` | Storage full |

## What had to change for this sensor

**15 enroll samples instead of 8.** With the driver's default of 8 the sensor
accepted the samples and the commit, but the stored print never matched: 0 of
7 valid reads, also after a cold boot. With 15 it matched on every valid read.
15 is what the community driver for the sibling sensor `04f3:0c6c` uses, taken
from a Windows capture of that device. Values other than 8 and 15 were not
tried.

**The first command after opening may be answered with six zero bytes.** With
a finger resting on the sensor while the device is opened (the driver resets
the device on open), the enrolled-count query got `00 00 00 00 00 00` on 0x83
about 140 ms after the reset, and the real reply never followed. The driver
now reads whole packets, drops a packet that is longer than the expected
reply, and re-sends this query after 100 ms.

**No abort when closing an idle device.** The original driver sent the abort
command on every close and released the interface without waiting for the
reply. Removed as unnecessary. It was suspected of causing the firmware hang
below; that turned out not to be the cause.

**Fixes that are not specific to this sensor.** Reporting of retryable errors
and of "not recognised" in verify/identify (a double free and a wrong result
code), no receive timeout while waiting for a finger, a crash when closing
right after a cancellation, and the storage feature declared in the driver
instead of patching the libfprint core.

**Left out.** The umockdev test data of the original branch was not carried
over, because it could not be run here.

## Firmware behaviour worth knowing

**Hang after an unanswered first query.** In 2 of 11 cases where the six zero
bytes appeared, the re-sent query was not answered either. The operation
failed after the 10 second receive timeout, and the USB reset of the next open
then failed with `device descriptor read/64, error -110` in the kernel log.
The sensor came back by itself after 60–75 seconds. One earlier hang looked
the same, before the stray-packet handling and the change to closing existed.
Forty quick open/close cycles without a finger never triggered it, with or
without the abort on close.

**Self-reset after an interrupted enrollment.** Once, about 100 seconds after
an enrollment ended in a timeout, the sensor disconnected and reconnected by
itself, and the print stored before was gone.

**Wipe is quick.** After `40 ff 99` the enrolled count was 0 within 0.7
seconds. The sibling `04f3:0c80` is reported to time out here; this sensor
does not.

**Duplicates are accepted.** The collision check reports a finger that is
already stored, but committing anyway works and uses the next slot. Identify
then kept reporting slot 0, so extra prints of one finger brought no
measurable gain.

**"Too little area" is frequent.** `0xfe` came back for 46–71 % of the touches
in the first sessions, during enrollment as well as verification, and fell to
roughly 1 in 6 with practice. The reply arrives as quickly as a successful
one, so it is not a timing issue on the host side.

## Open questions

- Does the status query (`40 ff 00`, ready = `0x03`) report "not ready" while
  a finger rests on the sensor after a reset? If so, polling it before the
  first command, as `elanmoc` does, should avoid the hang. The experimental
  patch does this; it has only been run with no finger on the sensor, where
  the first poll returns ready.
- Is the USB reset on open needed at all?
- The `04f3:0c6c` driver sends `00 10 <slot>` before the commit and an
  identify before the first enroll sample, both seen in a Windows capture of
  that device. Neither was needed here and neither was tried.
- `elanmoc` sends `40 ff 14 03` (set mode) on open and the sensor acknowledges
  it. `elanmoc2` never sets a mode. Whether the mode matters is unknown; the
  results above were the same before and after a cold boot.
- Listing and deleting single prints from fprintd. The older `elanmoc2-working`
  branch of the original driver has both.
