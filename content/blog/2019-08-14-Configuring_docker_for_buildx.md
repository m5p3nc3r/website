---
layout: post
title: Configuring Docker for buildx
---

Turns out the Yocto recipes were building docker version 18.09.3, but I need something greater that 19.03.0.  I was just about to rework the recipes to update them, but it turns our that the good folks upstream had already done this for me (thanks Bruce for [this patch](https://git.yoctoproject.org/cgit/cgit.cgi/meta-virtualization/commit/recipes-containers/docker?id=edd2454de498fc0f4d667408c3c61626a51ae6a3)).  A quick ```git pull``` in the ```meta-virtualization``` directory solved all my woes!

```bash
pushd <meta-virtualization directory>
git pull
popd
bitbake core-image-minimal
# Isn't Open Source Development Great!!!
```

copy the resulting image onto the sd-card... and ```docker version``` now reports 19.03.0-rc3!

Next we need to update the configuration to expose ```dockerd``` to the ethernet connection.  This is done by adding the following

```bash
cat << EOF > /etc/docker/daemon.json  
{
        "hosts": [
                "unix:///var/run/docker.sock",
                "tcp://192.168.7.2:2376"
        ],
        "experimental": true
}
EOF
```

It was at this point I noticed something wierd,  ```/etc/docker``` is a symlink to ```/run/docker``` which is a transient directory created in ramfs.  This means that each boot of the system resets the contents of ```/etc/docker``` so as it stands we can't keep any static configuration there.  The problem comes from [this line](https://git.yoctoproject.org/cgit/cgit.cgi/meta-virtualization/tree/recipes-containers/docker/docker_git.bb#n136), and I have a fix in the ```meta-usbbuilder``` project.  I plan on having a chat with the upstream maintainer about this to see if what is currently upstream is the intended behaviour.

Copy the resulting build onto the sd-card, reboot and you should now have the docker daemon listening for remote connections on the external TCP interface.

## Resizing the root filesystem

By default, the image written to the sd-card is only just bigger than the size of the files it needs to place on it.  This means that there is not much spare space left on the device.  We therefore need to expand the root partition.

There are two ways we can do this, firstly we could add some configuration to the projects ```local.conf``` to ask it to create a bigger image.  The problem with this would be that the time to copy it to the sd-card would be increased, and it still wouldn't necessarily make use of the whole space available.

The second approach would be to use ```resize2fs``` on the live partition.  If we mess this up, we risk corrupting the filsyste, but we can simply rewrite the root image to the sd-card to recover.  I will look at creating a script to do this automatically at some point.  This is the approach I will take here.

First we need to increase the size of the partition.

```bash
# Use fdisk to modify the partition tables
fdisk /dev/mmcblk0

# Print the current partiton list with the 'p' command
Command (m for help): p
Disk /dev/mmcblk0: 29.12 GiB, 31268536320 bytes, 61071360 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x97800000

Device         Boot Start    End Sectors   Size Id Type
/dev/mmcblk0p1       2048  34815   32768    16M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      40960 941813  900854 439.9M 83 Linux

# The important number is the 'Start' of the second partion, mine is
# 40960

# Delete the second partition
Command (m for help): d
Partition number (1,2, default 2): 2

Partition 2 has been deleted.

# Recreate the second partition at the same point as the old once
# Simply press 'return' when asked about the last sector, and keep
# the partition signature as it currently is
Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): 2
First sector (34816-61071359, default 34816): 40960
Last sector, +/-sectors or +/-size{K,M,G,T,P} (40960-61071359, default 61071359):

Created a new partition 2 of type 'Linux' and of size 29.1 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: n

# Next we have to write out the new partition table
Command (m for help): w

The partition table has been altered.
Syncing disks.
```

That's the scary bit over, now we simply have to resize the filesystem on the second partition and reboot.

```bash
# Resize the root fileystem
resize2fs /dev/mmcblk0p2
resize2fs 1.44.5 (15-Dec-2018)
Filesystem at /dev/mmcblk0p2 is mounted on /; on-line resizing required
EXT4-fs (mmcblk0p2): resizing filesystem from 450424 to 30515200 blocks
old_desc_blocks = 4, new_desc_blocks = 233
EXT4-fs (mmcblk0p2): resized filesystem to 30515200
The filesystem on /dev/mmcblk0p2 is now 30515200 (1k) blocks long.

# Reboot
reboot
```

You can see from the above log that the root filesystem on ```/dev/mmcblk0p2``` is now about 30G, which is much more healthy!

## Setting up the host buildx configuration

The device is now function how I wanted it, we need to do some quick configuration of docker on the host machine now.

```bash
# Create a build context called 'usb' and point it at our usb gadget
docker buildx create --name usb --driver docker-container --platform linux/arm64 tcp://192.168.7.2:2376

# Now tell docker to the newly created build context
docker buildx use usb
```

And that should be it...  We can test it by using the test application we created in a previous post [here](/Docker_buildx_from_scratch/#step-5---test).

```bash
docker buildx build --platform linux/arm64 -t m5p3nc3r/hello .
```

## Conclusion

From what I can see, it all seems to be working now.  Boots on its own, configures itself and makes the docker daemon available for remote access by the host PC, all without human interaction.

Next steps, which are also tracked [here](/todo/) will be

* Check the power consumption under load
* Make the root filesystem read only
  * Will ensure the root FS doesn't corrupt on power cycle
  * Can I use the host for storage?
* Strip the gadget kernel down to its bare minimum
  * improve boot time
  * decrease power consumption

But, I think the next task for me, which was suggested by [Carl Perry](https://www.packet.com/about/team/carl-perry/) from Packet after a conversation we had at the Arm Partner Meeting in Cambridge last week will be to try porting this to the new Raspberry PI 4 so I can make use of its faster, more efficient Arm Cortex A72 processors.  A quick check, and it looks like [meta-raspberrypi](http://git.yoctoproject.org/cgit.cgi/meta-raspberrypi) already has support for the Pi4, so how difficult can it be!
