# Maintainer: Natanael Copa <ncopa@alpinelinux.org>

_flavor=mptcp
pkgname=linux-${_flavor}
pkgver=5.4
case $pkgver in
	*.*.*)	_kernver=${pkgver%.*};;
	*.*) _kernver=$pkgver;;
esac
pkgrel=1
pkgdesc="Linux mptcp kernel"
url="https://www.kernel.org"
depends="mkinitfs"
_depends_dev="perl gmp-dev elfutils-dev bash flex bison"
makedepends="$_depends_dev sed installkernel bc linux-headers linux-firmware-any openssl-dev
	diffutils"
options="!strip"
_config=${config:-config-mptcp.${CARCH}}
install=
source="https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/linux-$_kernver.tar.xz
	mptcp-v5.4-b0b64e79bff7.patch
	0002-powerpc-config-defang-gcc-check-for-stack-protector-.patch

	config-mptcp.x86_64
	"
subpackages="$pkgname-dev:_dev:$CBUILD_ARCH"
_flavors=
for _i in $source; do
	case $_i in
	config-*.$CARCH)
		_f=${_i%.$CARCH}
		_f=${_f#config-}
		_flavors="$_flavors ${_f}"
		if [ "linux-$_f" != "$pkgname" ]; then
			subpackages="$subpackages linux-${_f}::$CBUILD_ARCH linux-${_f}-dev:_dev:$CBUILD_ARCH"
		fi
		;;
	esac
done

if [ "${pkgver%.0}" = "$pkgver" ]; then
	source="$source
	https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/patch-$pkgver.xz"
fi
arch="all !armhf"
license="GPL-2.0"

_carch=${CARCH}
case "$_carch" in
aarch64*) _carch="arm64" ;;
arm*) _carch="arm" ;;
mips*) _carch="mips" ;;
ppc*) _carch="powerpc" ;;
s390*) _carch="s390" ;;
esac

prepare() {
	local _patch_failed=
	cd "$srcdir"/linux-$_kernver
	if [ "$_kernver" != "$pkgver" ]; then
		msg "Applying patch-$pkgver.xz"
		unxz -c < "$srcdir"/patch-$pkgver.xz | patch -p1 -N
	fi

	# first apply patches in specified order
	for i in $source; do
		case $i in
		*.patch)
			msg "Applying $i..."
			if ! patch -s -p1 -N -i "$srcdir"/$i; then
				echo $i >>failed
				_patch_failed=1
			fi
			;;
		esac
	done

	if ! [ -z "$_patch_failed" ]; then
		error "The following patches failed:"
		cat failed
		return 1
	fi

	# remove localversion from patch if any
	rm -f localversion*
	oldconfig
}

oldconfig() {
	for i in $_flavors; do
		local _config=config-$i.${CARCH}
		local _builddir="$srcdir"/build-$i.$CARCH
		mkdir -p "$_builddir"
		echo "-$pkgrel-$i" > "$_builddir"/localversion-alpine \
			|| return 1

		cp "$srcdir"/$_config "$_builddir"/.config
		make -C "$srcdir"/linux-$_kernver \
			O="$_builddir" \
			ARCH="$_carch" \
			listnewconfig oldconfig
	done
}

build() {
	unset LDFLAGS
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
	for i in $_flavors; do
		cd "$srcdir"/build-$i.$CARCH
		make ARCH="$_carch" CC="${CC:-gcc}" \
			KBUILD_BUILD_VERSION="$((pkgrel + 1 ))-Alpine"
	done
}

_package() {
	local _buildflavor="$1" _outdir="$2"
	local _abi_release=${pkgver}-${pkgrel}-${_buildflavor}
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

	cd "$srcdir"/build-$_buildflavor.$CARCH
	# modules_install seems to regenerate a defect Modules.symvers on s390x. Work
	# around it by backing it up and restore it after modules_install
	cp Module.symvers Module.symvers.backup

	mkdir -p "$_outdir"/boot "$_outdir"/lib/modules

	local _install
	case "$CARCH" in
		arm*|aarch64) _install="zinstall dtbs_install";;
		*) _install=install;;
	esac

	make -j1 modules_install $_install \
		ARCH="$_carch" \
		INSTALL_MOD_PATH="$_outdir" \
		INSTALL_PATH="$_outdir"/boot \
		INSTALL_DTBS_PATH="$_outdir/boot/dtbs-$_flavor"

	cp Module.symvers.backup Module.symvers

	rm -f "$_outdir"/lib/modules/${_abi_release}/build \
		"$_outdir"/lib/modules/${_abi_release}/source
	rm -rf "$_outdir"/lib/firmware

	install -D include/config/kernel.release \
		"$_outdir"/usr/share/kernel/$_buildflavor/kernel.release
}

# main flavor installs in $pkgdir
package() {
	depends="$depends linux-firmware-any"

	_package mptcp "$pkgdir"
}

# subflavors install in $subpkgdir
virt() {
	_package virt "$subpkgdir"
}

_dev() {
	local _flavor=$(echo $subpkgname | sed -E 's/(^linux-|-dev$)//g')
	local _abi_release=${pkgver}-${pkgrel}-$_flavor
	# copy the only the parts that we really need for build 3rd party
	# kernel modules and install those as /usr/src/linux-headers,
	# simlar to what ubuntu does
	#
	# this way you dont need to install the 300-400 kernel sources to
	# build a tiny kernel module
	#
	pkgdesc="Headers and script for third party modules for $_flavor kernel"
	depends="$_depends_dev"
	local dir="$subpkgdir"/usr/src/linux-headers-${_abi_release}
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

	# first we import config, run prepare to set up for building
	# external modules, and create the scripts
	mkdir -p "$dir"
	cp "$srcdir"/config-$_flavor.${CARCH} "$dir"/.config
	echo "-$pkgrel-$_flavor" > "$dir"/localversion-alpine

	make -j1 -C "$srcdir"/linux-$_kernver O="$dir" ARCH="$_carch" \
		syncconfig prepare modules_prepare scripts

	# remove the stuff that points to real sources. we want 3rd party
	# modules to believe this is the soruces
	rm "$dir"/Makefile "$dir"/source

	# copy the needed stuff from real sources
	#
	# this is taken from ubuntu kernel build script
	# http://kernel.ubuntu.com/git/ubuntu/ubuntu-zesty.git/tree/debian/rules.d/3-binary-indep.mk
	cd "$srcdir"/linux-$_kernver
	find .  -path './include/*' -prune \
		-o -path './scripts/*' -prune -o -type f \
		\( -name 'Makefile*' -o -name 'Kconfig*' -o -name 'Kbuild*' -o \
		   -name '*.sh' -o -name '*.pl' -o -name '*.lds' -o -name 'Platform' \) \
		-print | cpio -pdm "$dir"

	cp -a scripts include "$dir"

	find $(find arch -name include -type d -print) -type f \
		| cpio -pdm "$dir"

	install -Dm644 "$srcdir"/build-$_flavor.$CARCH/Module.symvers \
		"$dir"/Module.symvers

	mkdir -p "$subpkgdir"/lib/modules/${_abi_release}
	ln -sf /usr/src/linux-headers-${_abi_release} \
		"$subpkgdir"/lib/modules/${_abi_release}/build
}

sha512sums="9f60f77e8ab972b9438ac648bed17551c8491d6585a5e85f694b2eaa4c623fbc61eb18419b2656b6795eac5deec0edaa04547fc6723fbda52256bd7f3486898f  linux-5.4.tar.xz
bcb5f45c83415df87e9b07056f996287205b357c2024ce59571411109fd61ed990423dcf169b4dd3c77b6cc1fa247f109ceee41ede0ef3a3693348ca1dcec346  mptcp-v5.4-b0b64e79bff7.patch
d19365fe94431008768c96a2c88955652f70b6df6677457ee55ee95246a64fdd2c6fed9b3bef37c29075178294a7fc91f148ead636382530ebfa822be4ad8c2f  0002-powerpc-config-defang-gcc-check-for-stack-protector-.patch
84624ae5ad050b9a362c1db2557234746bdcbc1e1336a260dbc40ccd7e7c15066fea6cda2deb9244a383da02e4f4994f9a9ac1e38297c47a6fba8900c1f028aa  config-mptcp.x86_64
01e82c077ada6d82f26b51f6739f89c85c7e0f9d4367f733d99f0d9a486de5f3090b2ae5de60957c9d51a56130de2dd034a83c292cb88adfb466238f90275ba9  patch-5.4.xz"
