# Maintainer: Jan Alexander Steffens (heftig) <heftig@archlinux.org>

pkgbase=linux-nrz
pkgver=6.19.14.nrz1
_zfsver="2.4.1"
pkgrel=1
pkgdesc='Linux'
url='https://github.com/archlinux/linux'
arch=(x86_64)
license=(GPL-2.0-only)
makedepends=(
  autoconf
  automake
  bc
  cpio
  gettext
  libelf
  libtool
  pahole
  perl
  pkgconf
  python
  rust
  rust-bindgen
  rust-src
  tar
  xz

)
options=(
  !debug
  !strip
)
_srcname=linux-${pkgver%.*}
_srctag=v${pkgver%.*}-${pkgver##*.}
source=(
  https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/${_srcname}.tar.{xz,sign}
  #$url/releases/download/$_srctag/linux-$_srctag.patch.zst{,.sig}
  config  # the main kernel config file
  "https://github.com/openzfs/zfs/releases/download/zfs-${_zfsver}/zfs-${_zfsver}.tar.gz"
)
validpgpkeys=(
  ABAF11C65A2970B130ABE3C479BE3E4300411886  # Linus Torvalds
  647F28654894E3BD457199BE38DBBDC86092693E  # Greg Kroah-Hartman
  83BC8889351B5DEBBB68416EB8AC08600F108CDF  # Jan Alexander Steffens (heftig)
)
# https://www.kernel.org/pub/linux/kernel/v6.x/sha256sums.asc
sha256sums=('cde8bf6739be4a0777fedbbba5330b8188c55680c45a922a4dfa289cbec6f185'
            'SKIP'
            #'57c22879f2228398564091db2ec9b186acbd56dfb0e1072f83418bfdd3829aae'
            #'SKIP'
            '31766d76d2384a385a4c30ccc894ad443065981e8c70ac6b70252ab3bf283f2d'
            'c17b69770f0023154f578eb8c7536a70f07d6a3bb0bd38f04fa0e8811c3c1390')
b2sums=('64c2a0003d8080f268772d36923ff6ef8b2d55320ea08b77ad39384c98c9a5c1a8e71425470619aa3aa4dda8941f46aa9da364748cfa8fd9f8507a5ddd7ac03a'
        'SKIP'
        #'8ece2f1b2fc6530cdd65e597141550c184089a206b9aa49cb9e46d61d2e7cf9c3f07f35ed523670d892aa7e62626644a5b1e98dd9c6acd824cb7ad3254c17665'
        #'SKIP'
        'cf8a4ef80e29129cd4c5ddd43d3e03fe33c5dc71455abf9710b49434c2aef2463acd7351e1eb59e480d98df83d560a0fc4121bb4dc82758b5e70ac9e5497c6d5'
        'dc7eedb989297ae13d42c1f7c3b55b393d47f1706a029c38801381675909069c6c2112ca545ac395cb2e7bcbaf225fa79982e3c6f3adeca0d30828b88da0f58b')

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

prepare() {
  cd $_srcname

  echo "Setting version..."
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    src="${src%.zst}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  echo "Setting arch to x86-64-v2"
  sed -i 's/march=x86-64 /march=x86-64-v2 /' arch/x86/Makefile

  echo "Setting config..."
  cp ../config .config
  make olddefconfig
  diff -u ../config .config || :

  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make all
  make KCFLAGS="-march=x86-64-v2 -mtune=generic" -C tools/bpf/bpftool vmlinux.h feature-clang-bpf-co-re=1

  cd "$srcdir"
  rm -rf "build-zfs"
  mkdir "build-zfs"
  cd "build-zfs"

  "$srcdir/zfs-${_zfsver}/autogen.sh"
  "$srcdir/zfs-${_zfsver}/configure" \
    --prefix=/usr \
    --sysconfdir=/etc \
    --sbindir=/usr/bin \
    --libdir=/usr/lib \
    --datadir=/usr/share \
    --includedir=/usr/include \
    --with-udevdir=/usr/lib/udev \
    --libexecdir=/usr/lib \
    --with-config=kernel \
    --with-linux="$srcdir/$_srcname" \
    --with-linux-obj="$srcdir/$_srcname"
  make
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(
    coreutils
    initramfs
    kmod
  )
  optdepends=(
    'linux-firmware: firmware images needed for some devices'
    'scx-scheds: to use sched-ext schedulers'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  provides=(
    KSMBD-MODULE
    NTSYNC-MODULE
    VIRTUALBOX-GUEST-MODULES
    WIREGUARD-MODULE
  )
  replaces=(
    virtualbox-guest-modules-arch
    wireguard-arch
  )

  cd $_srcname
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  # remove build link
  rm "$modulesdir"/build
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux tools/bpf/bpftool/vmlinux.h
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts
  ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"

  # required when STACK_VALIDATION is enabled
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # required when DEBUG_INFO_BTF_MODULES is enabled
  install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h
  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Installing Rust files..."
  #install -Dt "$builddir/rust" -m644 rust/*.rmeta
  #install -Dt "$builddir/rust" rust/*.so

  echo "Installing unstripped VDSO..."
  make INSTALL_MOD_PATH="$pkgdir/usr" vdso_install \
    link=  # Suppress build-id symlinks

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

package_zfs-linux-nrz() {
  pkgdesc="OpenZFS kernel modules for the $pkgbase kernel"
  license=(CDDL)
  depends=(
    kmod
    "$pkgbase=$pkgver-$pkgrel"
    "zfs-utils=${_zfsver}"
  )
  provides=(
    zfs
    spl
  )
  conflicts=(
    spl-dkms
    spl-dkms-git
    zfs-dkms
    zfs-dkms-git
    zfs-dkms-rc
  )

  cd "$srcdir/build-zfs/module"
  make DESTDIR="$pkgdir" INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  rm -rf "$pkgdir/usr/src"
}

package_zfs-linux-nrz-headers() {
  pkgdesc="OpenZFS build files for the $pkgbase kernel"
  license=(CDDL)
  depends=("$pkgbase-headers=$pkgver-$pkgrel")
  provides=(
    spl-headers
    zfs-headers
  )
  conflicts=(
    spl-headers
    zfs-headers
  )

  cd "$srcdir/build-zfs"
  make DESTDIR="$pkgdir" install

  rm -rf "$pkgdir/lib"

  sed -i "s|$srcdir||g" "$pkgdir/usr/src/zfs-${_zfsver}/$(<"$srcdir/$_srcname/version")/Module.symvers"
}

pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
  "zfs-linux-nrz"
  "zfs-linux-nrz-headers"
)
for _p in "${pkgname[@]}"; do
  [[ $_p == zfs-linux-nrz || $_p == zfs-linux-nrz-headers ]] && continue
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
