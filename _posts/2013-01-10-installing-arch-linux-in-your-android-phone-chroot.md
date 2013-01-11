---
layout: post
title: "Installing Arch Linux in your Android phone (chroot)"
description: ""
category: hack
tags: [android, archlinux]
---

{% include JB/setup %}
## Requirements

* A rooted phone

  See [http://ready2root.com](http://ready2root.com) if you have no idea how to do that.

* SSH root access to your phone

  I use [SSHDroid](https://play.google.com/store/apps/details?id=berserker.android.apps.sshdroid&feature=nav_result#?t=W251bGwsMSwyLDNd), they have a free edition in the Play store.
  SSHDroid comes with busybox, which is also required. You can also install it ([Busybox](https://play.google.com/store/apps/details?id=stericson.busybox&feature=search_result#?t=W251bGwsMSwxLDEsInN0ZXJpY3Nvbi5idXN5Ym94Il0.)) from the Play store separately.

* 1GB of free space in your SD card

  We'll create an image there to bootstrap and keep the Arch Linux filesystem.

* A Linux server/workstation/laptop

  We'll use some standard Linux tools to manipulate the official Arch Linux images and I used my Debian laptop for that.

Finally, I'd like to mention that I've tested this HOWTO 
using a [CyanogenMod ROM](http://cyanogenmod.com/) (versions 10.0 and 10.1) 
in my Samsung Galaxy S3 (I9300) and Galaxy S (I9000) phones, but it should work 
in any modern ROM as long as you have the requirements installed. YMMV.

The following Arch Linux ARM forum thread inspired the whole thing:

[http://archlinuxarm.org/forum/viewtopic.php?t=1361](http://archlinuxarm.org/forum/viewtopic.php?t=1361)

## What is this useful for?

It pretty much depends on you :). Your only limit is your imagination.

I use Arch in my phone to build ARM software, run ruby/mruby scripts, install VIM,
run MPD (the music player daemon streaming my music) and have real 
git-annex repositories.

Having a real Arch Linux filesystem with all the goodies, as opposed to the crippled one that comes with Android, is great for the power users (and that's why the Ubuntu phone will be great, BTW).

## Start your engines!

First, we need to download an Arch Linux image for ARM computers (~125 MB).

My galaxy S3 phone is an Exynos **ARMv7** phone, so I downloaded the
ODROID-X image, which should be optimized for my phone (just guessing though).

You can get it from:

[http://archlinuxarm.org/os/ArchLinuxARM-odroidx-latest.img.gz](http://archlinuxarm.org/os/ArchLinuxARM-odroidx-latest.img.gz)

<sub>
<b>Note:</b> ARMv5 images for older phones
<br/>
There's a more generic ARMv5 image that should run on older phones:
<br/>
http://archlinuxarm.org/os/ArchLinuxARM-armv5te-latest.tar.gz
<br/>
Note that the image format is different for ARMv5 since it's just a
tarball, so the process with this image is a little bit different.
</sub>

Download the image now, we'll need it to continue with the HOWTO.

## Extracting the filesystem from the image

<sub>
<b>Note:</b> skip to the next section if you are using the ARMv5 image. That image 
is ready to be uploaded to the phone.
</sub>

Since the official (RAW) image has been partitioned, the first thing we need to do
is to map the partitions to loop devices and extract the root filesystem
from it, creating a compressed tarball to upload to the phone.

Let's create temporary working directory first:

```
$ mkdir ~/tmp/archlinux
$ cd ~/tmp/archlinux
```

The image is compressed so, assuming it has been downloaded to 
the '~/downloads' directory, gunzip it first. The uncompressed image is
quite big (~3GB), so make sure you have enough free space to do it:

```
$ gunzip ~/downloads/ArchLinuxARM-odroidx-latest.img.gz
```

### Map the partitions
We have the uncompressed image in ~/downloads/ArchLinuxARM-odroidx-latest.img 
now. Create the device maps with kpartx so we can mount the root 
filesystem. We'll need to use 'sudo' for that, if we're not root. 
List the available partitions first (command output included):


```
# list partitions devmappings that would be added by -a
$ sudo kpartx -l ~/downloads/ArchLinuxARM-odroidx-latest.img
loop1p1 : 0 8191 /dev/loop1 1
loop1p2 : 0 106496 /dev/loop1 8192
loop1p3 : 0 6291456 /dev/loop1 114688
loop deleted : /dev/loop1

```

It looks like the root filesystem is in the thirt partition (the biggest one).
Add the partition devmappings now:

```
# add partition devmappings
$ sudo kpartx -a ~/downloads/ArchLinuxARM-odroidx-latest.img
# Double check they've been mapped
$ sudo losetup -a
/dev/loop0: [fe03]:5249110 (/home/rubiojr/downloads/ArchLinuxARM-odroidx-latest.img)
```

Good. You can find the mappings under /dev/mapper now:

```
ls /dev/mapper/
blueleaf-home  blueleaf-swap_1  loop0p1  loop0p3
blueleaf-root  control          loop0p2  sda5_crypt
```

### Mount the root partition and create the FS tarball

_/dev/mapper/loop0p3_ is the root filesystem, so let's mount it:


```
$ sudo mount /dev/mapper/loop0p3 /mnt/
$ ls /mnt/
bin   dev  home  lost+found  mnt  proc  run   srv  tmp  var
boot  etc  lib   media       opt  root  sbin  sys  usr
$ cat /mnt/etc/issue
Arch Linux \r (\l)
```

Yeah, that's our spiffy Arch Linux root filesystem.

Create the tarball now, we'll upload it to the phone SD card after that.
**It's importat** we create the tarball as root or using sudo to preserve 
the permissions and do it from the root of the filesystem. That is:


```
$ cd /mnt/
$ sudo tar czf /home/rubiojr/tmp/archlinux/archlinux.tar.gz .
$ ls -lh /home/rubiojr/tmp/archlinux/archlinux.tar.gz
-rw-r--r-- 1 root root 125M Jan 10 22:49 /home/rubiojr/tmp/archlinux/archlinux.tar.gz
```

Where _/home/rubiojr/tmp/archlinux_ is the temporary working directory we
created initially. Creating the tarball should take a while and produce a
~125MB file.

## Upload the tarball to the phone

Upload or copy it to the phone SD card or the built-in storage, 
if you prefer. Just copy it where you have enough storage to create 
the new (and big) root filesystem image that we we'll create.

I uploaded the tarball using SSH (since I have SSHDroid installed), but use 
whatever method you prefer.


## Onto the phone

Once the image has been uploaded, login to the phone via SSH and go to 
the root of the sdcard, where the tarball has been uploaded. Create a
temporary directory there:

```
$ cd /sdcard
$ mkdir archlinux
```

Create the (RAW) image that will keep the filesystem in the phone and 
format it:

```
# Create the 750MB image file
$ dd if=/dev/zero of=archlinux.img seek=749999999 bs=1 count=0 
$ ls -lh archlinux.img
----rwxr-x    1 system   sdcard_r  715.3M Jan 10 22:08 archlinux.img
# Format the image
$ mke2fs -F archlinux.img
```

<sub>
Why do we create the RAW image? Chances are that your SD card or internal 
storage is VFAT formatted, and VFAT does not support proper permissions, 
hardlinks and symlinks IIRC, so you'll see errors if you try to extract 
the filesystem tarball in a directory, instead of a EXT2 formatted, loopback
mounted image.
</sub>

Mount the image to the directory we created, so we can dump the contents 
of the tarball into it:

```
$ mount archlinux.img archlinux/
```
**Note**: some ROMs do not have loop devices support in the kernel, so the mount command above may not work\!. Modern CyanogenMod ROMs support it.

Double check it's mounted:

```
$ mount |grep archlinux
/dev/loop0 on /storage/sdcard0/archlinux type ext2 (rw,relatime,errors=continue)
```

Extract the contents of the tarball into the temporal directory we created.
Assuming you uploaded the tarball to /sdcard/archlinux.tar.gz:

```
$ cd archlinux
$ tar xzf /sdcard/archlinux.tar.gz
```

We should have an Arch Linux FS in /sdcard/archlinux now.

```
$ ls /sdcard/archlinux
bin         home        mnt         run         tmp
boot        lib         opt         sbin        usr
dev         lost+found  proc        srv         var
etc         media       root        sys
```

## Last steps

To have a fully functional chroot, we need to bind mount some filesystems
and create a proper resolv.conf, so we have proper name resolution.


```
$ mount -o bind /dev/ /sdcard/archlinux/dev/
$ mount -t proc proc /sdcard/archlinux/proc/
$ mount -t sysfs sysfs /sdcard/archlinux/sys/
$ mount -t devpts devpts /sdcard/archlinux/dev/pts/
$ echo "nameserver 8.8.8.8" > /sdcard/archlinux/etc/resolv.conf
```

Good! the last step is to upgrade your brand new Arch Linux chroot :)

```
$ chroot /sdcard/archlinux/ pacman -Syu # upgrades...
$ chroot /sdcard/archlinux/ /bin/bash
```

Enjoy!


P.S. Cleaning up the mess, umounting, unmapping devs, etc is left as an exercise for the reader.
