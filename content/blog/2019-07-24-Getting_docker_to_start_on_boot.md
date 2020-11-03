---
layout: post
title: Getting docker to start on boot
---

In this post, we look at what is needed to get docker running from initial boot of the device.  This is the first step in making the device self-contained.

## Docker needs a network

Docker really likes having a properly configured network before it starts, so all we need to do is make sure that the ```g_ether``` module loaded at boot time.  Luckily, the people over at _yocto towers_ have thought of this.  There are two ways to achieve this, the simple one where we simply add this to our ```local.conf``` file.

```bash
echo KERNEL_MODULE_AUTOLOAD += " g_ether" >> conf/local.conf
```

But this would mean thay anybody who wanted to build this project would need to add that configuration to their ```.local.conf``` as well, so my preferred approach is to add it to our pre-existing ```linux-stable_5.0.bbappend``` in the ```meta-usbbuilder``` project.  I have added this to the git repo [here](https://github.com/m5p3nc3r/meta-usbbuilder/blob/78c51a0177c07db8b9c6b4616038010940b0d880/recipes-kernel/linux/linux-stable_5.0.bbappend#L9).

The next step to to have the network auto-configure.  We need to edit the ```/etc/network/interfaces``` file to remove ```auto eth0``` to prevent the wired ethernet from coming up, and add ```auto usb0``` to bring up the gadget USB ethernet interface.  The best way to achieve this is with a ```do_install_append``` function added to an ```init-ifupdown_1.0.bbappend``` because the ```init-ifupdown``` package is responsible for adding the interfaces file to our image.

This has also been added to the ```meta-usbbuilder``` project and you can browse the changes [here](https://github.com/m5p3nc3r/meta-usbbuilder/tree/master/recipes-core/init-ifupdown).

In order to get the DNS entry being processed properly, we need to add the ```resolveconf``` package to our distro.  This has been done [here](https://github.com/m5p3nc3r/meta-usbbuilder/blob/d0edf0cc8bd9e876b92099faf59d84140d9535c9/recipes-core/core-image/core-image-minimal.bbappend#L3) by adding it to the ```core-image-minimal.bbappend```.

## Entropy - Entropy - Entropy

[![random](https://imgs.xkcd.com/comics/random_number.png)](https://xkcd.com/221/)

Experience tells me that the CPU backed random number entropy on the ODroid-c2 is not up to the job required by the container runtime, so we need to give it a little helping hand.  

```bash
# You can check your RNG Entropy by executing this command
cat /proc/sys/kernel/random/entropy_avail
37
```

anything less than ~200 is a problem.  I am showing 37!

[![dilbert random](https://assets.amuniversal.com/321a39e06d6401301d80001dd8b71c47)](https://dilbert.com/strip/2001-10-25)

So I have added the ```rng-tools``` package that enables the use of hardware random number generators that will improve the entropy hugely!  I followed the process outlined above by adding the package to the ```meta-usbbuilder``` [here](https://github.com/m5p3nc3r/meta-usbbuilder/blob/d0edf0cc8bd9e876b92099faf59d84140d9535c9/recipes-core/core-image/core-image-minimal.bbappend#L3)

There are also a couple of fixes in the runtime configuration in the overlay recipes found [here](https://github.com/m5p3nc3r/meta-usbbuilder/tree/master/recipes-support/rng-tools) that disables the jitter driver and fixes up an issue with the init script generation.

Rebuild ```core-image-minimal``` and push the resulting image on the sd-card and test.

```bash
# Test RNG Entropy once the device has been updated
cat /proc/sys/kernel/random/entropy_avail
4035
```

As you can see, my entropy is now 4035...  thats much more acceptable :¬)

## Conclusion time

After a bit of bitbake magic, we now have the gadget starting all the services I need at boot time, another step in the right direction.

Next steps will be to configure docker to accept remote connections, and in the back of my mind I am really wanting to move to systemd, but that probably says more about my mental health than anything else.
