# Maintainer: 7Ji <pugokughin@gmail.com>

_desc="AArch64 vendor kernel for Orange Pi 5/5B/5Plus/5Pro (git version)"
_pkgver_suffix=orangepi5
_pkgver_uname=${_pkgver_main}-${_pkgver_suffix}
_orangepi_repo=linux-orangepi
_srcname=${_orangepi_repo}
_pkgbase=linux-aarch64-${_pkgver_suffix}
pkgbase=${_pkgbase}-git
pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
)
pkgver=6.1.43.r1265811.752c0d0a12fd.609691c
pkgrel=1
arch=(aarch64)
_gh_ornagepi=https://github.com/orangepi-xunlong
url="${_gh_ornagepi}/${_orangepi_repo}"
license=(GPL2)
makedepends=( # Since we don't build the doc, most of the makedeps for other linux packages are not needed here
  'kmod' 'bc' 'dtc' 'uboot-tools' 'git'
)
options=(!strip !distcc)
source=(
  "git+${url}.git#branch=orange-pi-6.1-rk35xx"
  "git+${_gh_ornagepi}/orangepi-build.git#branch=next"
  'linux.preset'
  '02-downgrade-mali.patch'
)
sha256sums=(
  'SKIP'
  'SKIP'
  'bdcd6cbf19284b60fac6d6772f1e0ec2e2fe03ce7fe3d7d16844dd6d2b5711f3'
  '92170421688d30824249a7380009614c42faec21a98eea1001ba5f2f5d82de2f'
)

_config=external/config/kernel/linux-rockchip-rk3588-current.config

prepare() {
  echo "Setting version..."
  cd orangepi-build
  local rev_config="$(git rev-list --count HEAD "${_config}")"
  local id_config="$(git rev-parse --short HEAD:"${_config}")"
  cd ../"${_srcname}"
  local rev_kernel="$(git rev-list --count HEAD)"
  local id_kernel="$(git rev-parse --short HEAD)"
  scripts/setlocalversion --save-scmversion

  echo - > localversion.09-hyphen
  echo "r$(( "${rev_kernel}" + "${rev_config}" ))" > localversion.10-release-total
  echo - > localversion.19-hyphen
  echo "${id_kernel}" > localversion.20-id-kernel
  echo - > localversion.29-hyphen
  echo "${id_config}" > localversion.30-id-config
  echo "-${pkgrel}" > localversion.40-pkgrel
  echo "${pkgbase#linux}" > localversion.50-pkgname

  echo "Patch downgrade-mali.patch to fix strict warning..."
  patch -p1 -N -i ../02-downgrade-mali.patch || true

  echo "Updating config file..."
  cat "../orangepi-build/${_config}" > '.config'
  make olddefconfig
}

pkgver() {
  cd "${_srcname}"
  printf '%s.%s.%s.%s' \
    "$(make kernelversion)" \
    "$(<localversion.10-release-total)" \
    "$(<localversion.20-id-kernel)" \
    "$(<localversion.30-id-config)"
}

build() {
  cd "${_srcname}"

  # get kernel version, which will be used later for modules
  make prepare
  make -s kernelrelease > version

  # Host LDFLAGS or other LDFLAGS set by makepkg/user is not helpful for building kernel: it should links nothing outside of itself
  unset LDFLAGS
  # Only need normal Image, as most Rockchip devices does not need/support Image.gz
  # Image and modules are built in the same run to make sure they're compatible with each other
  # -@ enables symbols in dtbs, so overlay is possible
  make ${MAKEFLAGS} DTC_FLAGS="-@" Image modules dtbs
}

_package() {
  pkgdesc="The Linux Kernel and module - ${_desc}"
  depends=(
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'uboot-legacy-initrd-hooks: to generate uboot legacy initrd images'
    'linux-firmware: firmware images needed for some devices'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  backup=(
    "etc/mkinitcpio.d/${pkgbase}.preset"
  )

  cd "${_srcname}"

  # Install modules
  echo "Installing modules..."
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # Install DTBs, not to target pkg, but in srcdir, so the later package() routine could use them
  make INSTALL_DTBS_PATH="${srcdir}/dtbs" dtbs_install

  # Install pkgbase
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # Install kernel image (this is technically not vmlinuz, but I name it this way to utilize mkinitcpio's existing hooks)
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # Remove hbuild and source links, which points to folders used when building (i.e. dead links)
  rm -f "${_dir_module}/"{build,source}

  # install mkinitcpio preset file
  sed "s|%PKGBASE%|${pkgbase}|g" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # Install DTB
  echo 'Installing DTBs for Rockchip SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/rockchip"
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
  # Compiling modules against the tree needs gcc-wrapper.py clang-wrapper.py,
  # which both need Python.
  # Discussed in https://github.com/7Ji/archrepo/issues/5
  # About why depends instead of optdepends, this is a similar decision to
  # https://bugs.archlinux.org/task/69654
  depends=('python')
  
  # Mostly copied from alarm's linux-aarch64 and modified
  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "${_builddir}" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile
  install -Dt "${_builddir}/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "${_builddir}" -a scripts

  echo "Installing headers..."
  cp -t "${_builddir}" -a include
  cp -t "${_builddir}/arch/arm64" -a arch/arm64/include
  install -Dt "${_builddir}/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s


  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "${_builddir}/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "${_builddir}/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} = */arm64/ ]] && continue
    echo "Removing $(basename "${_arch}")"
    rm -r "${_arch}"
  done

  echo "Removing documentation..."
  rm -r "${_builddir}/Documentation"

  echo "Removing broken symlinks..."
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "${_builddir}" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v ${STRIP_SHARED} "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v ${STRIP_STATIC} "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v ${STRIP_BINARIES} "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v ${STRIP_SHARED} "$file" ;;
    esac
  done < <(find "${_builddir}" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "${_builddir}/vmlinux"

  echo "Adding symlink..."
  mkdir -p "${pkgdir}/usr/src"
  ln -sr "${_builddir}" "$pkgdir/usr/src/$pkgbase"
}

for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done
