# libfprint with the elanmoc2 driver for the ELAN 04f3:0c77 fingerprint reader
# (ASUS ExpertBook B9403CVAR and others). See README.md.
#
# The package is named libfprint-git so that Omarchy's fingerprint setup
# (omarchy-setup-security-fingerprint) accepts it as is. The epoch keeps a
# system update from replacing it with the distribution's own libfprint-git.
# On plain Arch it replaces the libfprint package.
pkgname=libfprint-git
pkgver=1.94.100.r10.g6f9479c
pkgrel=1
epoch=2
pkgdesc="Library for fingerprint readers, with the elanmoc2 driver for ELAN 04f3:0c77"
arch=(x86_64)
url="https://fprint.freedesktop.org/"
license=(LGPL-2.1-or-later)
depends=(gcc-libs glib2 glibc libgudev libgusb openssl pixman)
makedepends=(git meson glib2-devel)
provides=("libfprint=${pkgver%%.r*}" libfprint-2.so=2-64)
conflicts=(libfprint)
options=(!debug)
_commit=6f9479c3d55f847c1b3769f28ceb99227f9858cf
source=("libfprint::git+https://gitlab.freedesktop.org/libfprint/libfprint.git#commit=${_commit}"
        0001-elanmoc2-driver-with-04f3-0c77.patch)
sha256sums=('SKIP'
            'c8a3fa9b85ef27c4eaaf70362682e20f933cb51f693d3534b1cc27ad97047db6')

prepare() {
  cd libfprint
  patch -Np1 -i ../0001-elanmoc2-driver-with-04f3-0c77.patch
}

build() {
  meson setup libfprint build \
    --prefix=/usr --libexecdir=lib --sbindir=bin --buildtype=plain \
    --wrap-mode=nodownload -Db_pie=true \
    -Dintrospection=false \
    -Ddoc=false \
    -Dgtk-examples=false \
    -Dinstalled-tests=false
  meson compile -C build
}

package() {
  meson install -C build --destdir "$pkgdir"
}
