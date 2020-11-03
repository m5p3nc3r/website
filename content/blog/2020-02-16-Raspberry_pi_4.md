---
layout: post
title: Let's try this on a Raspberry Pi 4
---

Its been a while since my last post.  I'm going to put some of the blame on the Raspberry Pi not having a functional serial console  from the get go, and the rest on work/life/...

![Raspberry PI 4 connected via USB C alone](/images/rpi4.jpg)

I finally have the the builder project working on the Raspberry Pi 4, things became a lot easier once I had added configuration for a serial console over the USB interface.  The recipes for this can be found [here](https://github.com/m5p3nc3r/meta-usbbuilder/tree/master/recipes-usbbuilder/init-usbgadget).  I used the libcomposite kernel module to achieve this, currently tested on Linux, Windows and Mac, and the serial interface is working fine.  Please feel free to add this to your project, its really quite nice having a single cable connected to the Pi4.

Next step was to add a network interface using the libcomposite framework.  I must admit that I only have this working with a Linux host at the moment.  I believe I need an RNDIS interface for Windows which is simple to add, and Mac is a quandary for me (as I don't have access to a Mac for development).  If anyone can help with this it would be appreciated.

I need to understand how the configuration profiles work for USB gadgets, as I believe this can be a way to express the same interface config to the host irrespective of the host OS.

## Building the project

Sources for the OpenEmbedded recipes can be found [here](https://github.com/m5p3nc3r/meta-usbbuilder).  All of this has been tested against the Zeus release of Yocto.  So please make sure you checkout the Zeus branch of each of the the dependant projects.

```bash
## ######################################
## Get all the required source recipes
## ######################################
git clone https://github.com/m5p3nc3r/meta-usbbuilder.git
git clone git://git.yoctoproject.org/poky.git && pushd poky && git checkout zeus && popd
git clone https://git.yoctoproject.org/git/meta-raspberrypi && pushd meta-raspberrypi && git checkout zeus && popd
git clone git://git.openembedded.org/meta-openembedded && pushd meta-openembedded && git checkout zeus && popd
git clone https://git.yoctoproject.org/git/meta-virtualization && pushd meta-virtualization && git checkout zeus && popd

# Create a build directory for our Pi4 profile
source poky/oe-init-build-env build_pi4

# We should now be in the <project>/build_pi4 directory
# Add our configuration options...

cat <<EOF >> conf/local.conf
# I like to keep a download cache for all projects
DL_DIR = "\${TOPDIR}/../downloads"
SSTATE_DIR ?= "\${TOPDIR}/../sstate-cache"

# Configure the target machine
MACHINE = "raspberrypi4-64"

# Ensure the kernel modules are added (for test/debug purposes - can remove later)
CORE_IMAGE_EXTRA_INSTALL += " kernel-modules"

# Enable USB gadget mode
ENABLE_DWC2_PERIPHERAL = "1"

# I think there is a bug in the u_serial driver where its not showing up in
# /proc/consoles with the version of the kernel used by the RPI.
# Hopefully can remove this when we move to 5.5?
SYSVINIT_ENABLED_GETTYS = "GS0"
SERIAL_CONSOLES = "115200;ttyAMA0"
EOF

# Now we need to add the external recipes to our bblayers.conf
bitbake-layers add-layer ../meta-raspberrypi
bitbake-layers add-layer ../meta-usbbuilder
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-openembedded/meta-filesystems
bitbake-layers add-layer ../meta-openembedded/meta-python
bitbake-layers add-layer ../meta-openembedded/meta-multimedia
bitbake-layers add-layer ../meta-openembedded/meta-networking
bitbake-layers add-layer ../meta-virtualization

# Now lets build the base image
bitbake core-image-minimal
```

## Install the image on the sd card

I'm going to assume you know how to use dd here.  The SD card I am using gets mounted on ```/dev/mmcblk0```, yours may mount somewhere else.  ```lsblk``` can help you here.  Please replace ```/dev/mmcblk0``` in the next step with the correct location of your SD card, otherwise you may end up writing the root filesystem for the Pi4 over something that you care about!

```bash
# Fedora likes to auto-mount any removable media you install, so lets unmount it
umount /dev/mmcblk0*
sudo dd bs=1M if=tmp/deploy/images/raspberrypi4-64/core-image-minimal-raspberrypi4-64.rpi-sdimg of=/dev/mmcblk0
sync
```

## Boot and connect to the Pi4

Pop the SD card into your Pi4 and boot.  Now I'm a bit old school here, so I'll be using minicom to connect from the host.  You can use whatever terminal emulator you like, the important things are that the baud rate is 115200 8N1 with no flow control.  Minicom already has a configuration for ttyAMA0 on my system, so all I need to do now is

```bash
minicom -o ttyACM0
```

## Resizing the root filesystem to use the whole disk

I have covered this in a previous post. All of the tools you need are included in the root filesystem, so check out the section [here](/Configuring_docker_for_buildx/#resizing-the-root-filesystem) which covers resizing the root filesystem and setting up the host docker configuration to make use of the Pi4 usb builder/

## Configuring the host network

Err... Will come to this in a future post, but the Pi4 has an IP address of 192.168.7.2/24 and is expecting the host to have an address of 192.168.7.1/24 with IP masquerading to the outside world.  If this is configured correctly, and you can test this by ```ping www.google.com``` on the Pi4, you are ready to play :¬)

## We are ready to play

Docker should now be fully operational on the Pi4, and exposing an IP interface over USB.  You can now explore using ```docker buildx``` for remote builds as shown in previous posts.  You should also be able to use this as a node in a compute cluster using tech like [k8s](https://kubernetes.io/), [k3s](https://k3s.io/) or [swarm](https://docs.docker.com/engine/swarm/).  This is where the fun begins!

