# libfprint driver for the ELAN 04f3:0c77 fingerprint reader

A patch that makes the ELAN `04f3:0c77` ("ELAN:ARM-M4") match-on-chip
fingerprint reader work with [libfprint](https://fprint.freedesktop.org/) and
fprintd, plus an Arch/Omarchy package build for it.

The reader is not supported by any libfprint release. Upstream requests:
[#447](https://gitlab.freedesktop.org/libfprint/libfprint/-/issues/447) (open
since 2021), [#510](https://gitlab.freedesktop.org/libfprint/libfprint/-/issues/510)
and [#692](https://gitlab.freedesktop.org/libfprint/libfprint/-/issues/692).

## Status

Working, with one known problem. Tested on a single machine.

| | Result |
|---|---|
| Enroll (15 samples) | Works |
| Verify / identify through fprintd | Works: 38 of 38 normal touches matched in the last two test runs |
| Other fingers | Rejected in every test where the finger was noted (6 of 6); no false match seen |
| Delete one print, clear all prints | Work |
| Finger resting on the sensor while the device is opened | Fails in about 1 of 10 cases, see [Known problems](#known-problems) |
| Omarchy's fingerprint setup (`omarchy-setup-security-fingerprint`) | Completed: enroll, verify and PAM configuration |
| Lock screen | Unlocks with the fingerprint |
| sudo, polkit | Configured by the setup; not exercised yet |
| Suspend/resume | Not tested yet |

Test machine: ASUS ExpertBook B9403CVAR, reader firmware 0x316 (USB
`bcdDevice` 3.16), Omarchy (Arch Linux), kernel 7.2.15, fprintd 1.94.5,
libfprint upstream commit `6f9479c3` (v1.94.100 + 10).

The same USB ID is reported in the ASUS ExpertBook B5 Flip and B7 Flip. Those
are untested with this patch.

## What is in here

| File | What it is |
|---|---|
| `0001-elanmoc2-driver-with-04f3-0c77.patch` | The driver, as one patch against upstream libfprint `6f9479c3` |
| `PKGBUILD` | Builds libfprint with the patch as an Arch package |
| `experimental/0002-wait-for-ready-on-open.patch` | Untested attempt at the known problem; not applied by default |
| `docs/protocol-notes.md` | What was learned about the sensor's protocol and firmware behaviour |

The driver is Davide Depau's `elanmoc2` driver, ported to current libfprint
and adjusted for this sensor. The upstream `elanmoc` driver does not work for
`04f3:0c77` (it opens the device, but its re-enroll check is never answered),
and neither does the older `elan` image driver.

## Install on Arch Linux or Omarchy

Build requirements: `base-devel`, `git`, `meson`, `glib2-devel`.

```sh
git clone https://github.com/inumineq/libfprint-elan-0c77.git
cd libfprint-elan-0c77
makepkg -si
sudo pacman -S --needed fprintd
```

The package is called `libfprint-git` and replaces `libfprint`. It carries
epoch 2 so that a system update does not swap it for the distribution's own
`libfprint-git`. The price is that you do not get newer libfprint versions
until you rebuild this one.

Then enroll and test, before touching PAM:

```sh
sudo fprintd-enroll -f right-index-finger "$USER"
fprintd-verify
```

Enrolling takes 15 good touches. Expect a fair number of "remove and retry"
prompts at first; they mean the sensor saw too little of the finger. Cover the
whole sensor with the pad of the finger and vary the position a little between
touches.

### PAM

Only after `fprintd-verify` works reliably for you.

On the test machine Omarchy's setup ran through without changes to the script,
and the lock screen unlocked with the fingerprint afterwards. Lift the finger
between touches and do not rest it on the sensor while the setup starts; see
[Known problems](#known-problems).

- **Omarchy:** `omarchy-setup-security-fingerprint` sets up sudo, polkit and
  the lock screen. It always enrolls the right index finger itself, so remove
  an existing print first with `sudo fprintd-delete "$USER"`.
  `omarchy-remove-security-fingerprint` undoes it (and removes the packages).
- **Arch:** follow the [fprint page](https://wiki.archlinux.org/title/Fprint)
  on the Arch wiki.

The password keeps working either way.

### Other distributions

Apply the patch to libfprint at commit
`6f9479c3d55f847c1b3769f28ceb99227f9858cf` and build it the usual way:

```sh
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git
cd libfprint
git checkout 6f9479c3d55f847c1b3769f28ceb99227f9858cf
patch -Np1 -i /path/to/0001-elanmoc2-driver-with-04f3-0c77.patch
meson setup build && meson compile -C build
```

This has only been built on Arch.

### Going back

```sh
sudo pacman -S libfprint        # Arch
sudo pacman -S libfprint-git    # Omarchy; pacman warns about the downgrade
```

## Known problems

**A finger resting on the sensor while the device is opened.** The sensor then
answers the first command with six zero bytes instead of a reply. The driver
asks again, which worked in 9 of 11 observed cases. In the other 2 the sensor
did not answer at all: the operation failed after a 10 second timeout, and the
USB reset of the next open left the sensor dead on the bus for about a minute.
It came back by itself every time, and the password works meanwhile.

This matters if the reader is built into the power button (as reported for the
ExpertBook B5 Flip): waking the machine with the button puts a finger on the
sensor at about the moment the lock screen opens the reader.

`experimental/0002-wait-for-ready-on-open.patch` polls the sensor's status
until it reports ready before using it, as the upstream `elanmoc` driver does.
It behaves correctly with no finger on the sensor and has never been tried in
the situation it is meant to fix.

**Too little finger area.** The sensor rejects a touch before matching when it
sees too little of the finger. In the first sessions this hit 46–71 % of the
touches; with practice it dropped to roughly 1 in 6. fprintd just asks for
another touch.

**Unexplained "no match" results.** The fprintd test loops produced 9 "no
match" results in 137 runs. Four came from a loop that deliberately included
other fingers. For the other five the finger on the sensor was not recorded,
so the rate of false rejections is somewhere between 0 and 5 in 118.

**An interrupted enrollment can cost the stored prints.** Once, an enrollment
that ended in a timeout was followed by the sensor restarting itself, and the
print stored before it was gone. Re-enroll if that happens.

**Not covered.** Listing the prints stored on the sensor and deleting a print
through fprintd are not implemented in this version of the driver;
`fprintd-delete` removes fprintd's copy only. The driver deletes a stored
print itself when the same finger is enrolled again, and fprintd clears the
sensor before the first enrollment.

## Test knobs

For experiments only. Set them in the environment of the program using
libfprint (for fprintd, a systemd drop-in).

| Variable | Effect |
|---|---|
| `ELANMOC2_ENROLL_SAMPLES=<8-40>` | Number of samples the print is built from (default for `04f3:0c77`: 15) |
| `ELANMOC2_ALLOW_DUPLICATES=1` | Store another print of a finger that is already enrolled instead of replacing it |

Several prints of the same finger did not measurably help here: the sensor
always reported the first one as the match.

## Credits

- Davide Depau wrote the `elanmoc2` driver
  ([libfprint fork](https://gitlab.freedesktop.org/Depau/libfprint), branch
  `elanmoc2`, and [MR !330](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/330))
  and the [Python proof of concept](https://github.com/depau/elanpoc) that it
  grew out of. This patch is his driver at commit `11f0316d` plus the changes
  listed in `docs/protocol-notes.md`.
- A user report on that proof of concept
  ([issue #2](https://github.com/depau/elanpoc/issues/2)) was the first sign
  that `04f3:0c77` speaks this protocol.
- Prashant Adhikari's
  [driver for 04f3:0c6c](https://github.com/ITx-prash/libfprint-elan-04f3-0c6c),
  another ARM-M4 sensor, pointed to using 15 enroll samples.
- The [linux-fingerprint-drivers](https://github.com/jedbillyb/linux-fingerprint-drivers)
  hub collected the leads for the ELAN match-on-chip family.

Developed with the help of Claude, Anthropic's AI model, which wrote the
changes while the owner of the laptop ran the tests on the hardware.

## License

LGPL-2.1-or-later, the same as libfprint. See `LICENSE`.
