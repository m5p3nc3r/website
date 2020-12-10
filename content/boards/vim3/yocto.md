---
layout: post
title: Yocto
date: 2020-10-24
---

## Yocto support for the Khadas Vim3

Board support is given through [meta-meson](https://github.com/superna9999/meta-meson)

My downstream fork is available [here](https://github.com/m5p3nc3r/meta-meson)

The aim is to have all modifications merged against the upstream.

## Upstream activities

* [x] [Moving the kernel recipes to use the proper linux-yocto framework](https://github.com/superna9999/meta-meson/pull/85)
* [x] [Add HDMI output for linux-yocto](https://github.com/superna9999/meta-meson/pull/87)
* [x] [Fixup linux-yocto to build for -sdboot targets](https://github.com/superna9999/meta-meson/pull/97)
* [x] [Add PCIe support for NVMe devices](https://github.com/superna9999/meta-meson/pull/98)
* [x] [Remove dupliacte include of linux-yocto-virtualization.inc](https://github.com/superna9999/meta-meson/pull/99)
* [x] [Use .dtb instead of .dts in boot config](https://github.com/superna9999/meta-meson/pull/100)

## Yocto configuration

```bash
# For development
git clone git@github.com:m5p3nc3r/meta-meson.git
# For production use the upstream
#git clone https://github.com/superna9999/meta-meson.git

# poky@master
git clone https://git.yoctoproject.org/git/poky
# We need to be on the master branch because the PCI enablement is
# in Linux 5.8 or later and dunfell's linux-yocto is 5.4
pushd poky && git checkout master && popd

# meta-openembedded@dunfell
git clone https://github.com/openembedded/meta-openembedded.git
pushd meta-openembedded && git checkout dunfell && popd

# meta-virtualization@dunfell
git clone https://git.yoctoproject.org/git/meta-virtualization
pushd meta-virtualization && git checkout dunfell && popd
```

You need to create a file meta-virtualization/recipes-kernel/linux/linux-yocto_5.8_virtualization.inc with the following contents because of the mismatch between poky@master and meta-virtualization@dunfell

```bash
# include the baseline meta virtualization configuration options
# after this include, we can do version specific things

include linux-yocto_virtualization.inc
```

Setup the build environment

```bash
source poky/oe-init-build-env build-vim3
```

Configure our build by adding the following to the end of the conf/local.conf

```bash
MACHINE = "khadas-vim3"
# linux-yocto is not the default kernel, select it here
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
# We need kernel changes for containers, this pulls them in
DISTRO_FEATURES_append = " virtualization"

# Kernel modules are needed for the container runtimes
IMAGE_INSTALL_append = " docker bash kernel-modules dropbear"
# These dependencies are just for development
IMAGE_INSTALL_append = " git e2fsprogs-mke2fs e2fsprogs-resize2fs"

# Update the storage location of share assets to speed up the build
DL_DIR = "${TOPDIR}/../downloads"
SSTATE_DIR = "${TOPDIR}/../sstate-cache"
```

Build the image

```bash
bitbake core-image-minimal
```

Burn the resulting image onto an sd-card

```bash
sudo dd status=progress conv=fsync \
        if=tmp/deploy/images/khadas-vim3/core-image-minimal-khadas-vim3.wic \
        of=/dev/mmcblk0 bs=1M
```

## Workarounds

The current build does not have NVMe support in the devicetree configuration.  This needs to be enabled by use of an overlay or modifying the devicetree manually in uboot using the ```fdt``` command.  The problem is the default boot flow used by the meta-meson recipes uses syslinux configuration and I can't find a sensible way of doing this with that boot flow.

Workaround for this is to remove the default boot command in ```/boot/syslinux/xxx.conf``` and create a properly signed ```/boot/boot.cfg```.  The default boot flow will pick up this new config and we can put our ```fdt``` commands here.

```bash
# Mount the root filesystem
sudo mount /dev/mmcblk0p1 root

# Create the boot script
cat << EOF  > /root/boot/boot.txt
setenv kernel_addr "0x11000000"
setenv dtb_addr "0x1000000"

setenv bootargs "console=ttyAML0,115200n8 root=/dev/mmcblk0p1 rootwait"
setenv kernel   "ext4load mmc 1 \${kernel_addr} /boot/fitImage"
setenv dtb      "ext4load mmc 1 \${dtb_addr} /boot/meson-g12b-a311d-khadas-vim3.dtb"
setenv fixfdt   "fdt addr \${dtb_addr} 0x10000; fdt set /soc/pcie status \"okay\""

setenv bootseq "bootm \${kernel_addr} - \${dtb_addr}"

setenv bootcmd "\${kernel}; \${dtb}; \${fixfdt}; \${bootseq}"

run bootcmd
EOF

# Sign the boot script
sudo mkimage -A arm -T script -O linux -d root/boot/boot.txt root/boot/boot.scr

# Unmount the root filesystem
sudo umount root
```
