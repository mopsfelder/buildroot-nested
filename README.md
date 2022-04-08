This repository contains Buildroot configuration files to build minimal guest
images for L1 and L2.  Guests are based on pSeries, 64-bit, little endian.

Guests are busybox-based and with minimal tweaks.  The main difference is that
L1 image has QEMU (target ppc64le-softmmu) in it.  Because of that, L1 image is
larger (300MB) than L2 (150M).  L1 image has QEMU and some spare space to copy
L2 image into it.  Both images have OpenSSH server and client binaries.  And are
built with e1000 network driver built-in in the kernel.

The build process is supposed to be executed on an x86_64 host, e.g.: your
laptop.  Tested on these host scenarios:

- Fedora 35 x86_64
- Ubuntu 20.04.3 LTS x86_64

The Buildroot configuration is set to use an external toolchain.  It uses
pre-built cross-compilers from Bootlin for x86_64 with code generation targeted
for ppc64le.  Running on a ppc64le host is possible with some adaption, e.g.:
changing Buildroot toolchain to use internal toolchain.

Required disk space: 8GB for each image.

# Instructions to build L1 and L2 images

## Clone repos

``` shell
git clone https://github.ibm.com/muriloo/buildroot-nested.git
git clone git://git.buildroot.net/buildroot l1-buildroot
git clone git://git.buildroot.net/buildroot l2-buildroot
```

## Install build deps for Buildroot

- Fedora

``` shell
sudo dnf install bash bc binutils bzip2 cpio file g++ gcc gzip make patch perl rsync sed tar unzip wget which zlib-devel
```

- Ubuntu

``` shell
sudo apt-get install bash bc binutils build-essential bzip2 cpio debianutils file g++ gcc gzip make patch perl rsync sed tar unzip wget zlib1g-dev
```

## Build **L1** image

``` shell
cd l1-buildroot/
git checkout 2022.02
make BR2_EXTERNAL=../buildroot-nested/l1 qemu_ppc64le_pseries_defconfig
make source
make all
```

## Build **L2** image

``` shell
cd l2-buildroot/
git checkout 2022.02
make BR2_EXTERNAL=../buildroot-nested/l2 qemu_ppc64le_pseries_defconfig
make source
make all
```

# Run guests

Make sure L0 host has `qemu-system-ppc64` built with target `ppc64-softmmu` enabled.

# Run **L1** guest

On L0 host:

``` shell
qemu-system-ppc64 -nographic -serial mon:stdio -M pseries,accel=kvm,cap-nested-hv=on -m 512 -kernel vmlinux -append "console=hvc0 rootwait root=/dev/sda" -drive file=rootfs.ext2,if=scsi,index=0,format=raw
```

Note: Copy L2 files into the L1 instance, e.g.: `scp root@<builder>:l2-buildroot/output/images/"*" .`

# Run **L2** guest

On L1 instance:

``` shell
qemu-system-ppc64 -nographic -serial mon:stdio -M pseries,accel=kvm,cap-nested-hv=off -m 256 -kernel vmlinux -append "console=hvc0 rootwait root=/dev/sda" -drive file=rootfs.ext2,if=scsi,index=0,format=raw
```
