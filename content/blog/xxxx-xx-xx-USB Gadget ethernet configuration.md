---
layout: post
title: USB Gadget ethernet configuration
draft: true
---

In the previous post, the network configuration was a very manual process.  In this post, I will be exploring how to make this an automatic 'plug and play' experience.

First we need to make sure that the usb gadget drive is loaded when the system boots.  To do this, we simply need to add make sure ```g_ether``` is added to ```/etc/modules```.  Mine now looks like this
```
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.

g_ether
```

Next, we are going to give our device a proper name so that other machines can refer to it by name.  I am going to call mine ```armbuilder```

    hostnamectl set-hostname armbuilder


We also need to make sure that we can resolve our own name, so edit ```/etc/hosts``` to have a line that looks like this.

    127.0.0.1    localhost armbuilder

Then we need to get the device configuring the network interface automatically.  To do this we are going to make use of [avahi](https://www.avahi.org/).

    apt-get install avahi-daemon avahi-autoipd libnss-mdns

TO get the usb ethernet interface using the tools correctly, create a file ```/etc/network/interfaces.d/usb0```, with the contents

    auto usb0
    iface usb0 inet ip4all

This will ensure that the usb device configures itself with the avahi autoip daemon.

Because the ODroid C2 also has a wired ethernet port that is configured by default, we can diable this by removing the file ```/etc/network/interfaces.d/eth0```

## Making this work with the host PC
