---
layout: post
title: "Building mruby-head with your Android Phone"
description: ""
category: hack
tags: [mruby, ruby, android, archlinux]
---
{% include JB/setup %}

Building mruby-head for ARM computers with your Android phone using a pre-made
Arch Linux chroot for ARMv7 phones (which includes the required tools).

See [Installing Arch Linux in your Android phone](http://rubiojr.rbel.co/hack/2013/01/10/installing-arch-linux-in-your-android-phone-chroot/) if you are
interested in the gory details.

## Requisites

* A rooted phone

  See [http://ready2root.com](http://ready2root.com) if you have no idea 
  how to do that.

* SSH root access to your phone, like [SSHDroid](https://play.google.com/store/apps/details?id=berserker.android.apps.sshdroid&feature=nav_result#?t=W251bGwsMSwyLDNd)

## Setup the chroot

SSH to your android phone and change directory to the root of the SD card:

    ssh root@my-android-ip
    cd /sdcard

Get the chroot image and chroot to it:

    mkdir mruby-build
    # md5sum http://download.frameos.org/chroots/mruby-build-chroot.img.tar.gz.md5
    wget http://download.frameos.org/chroots/mruby-build-chroot.img.tar.gz
    tar xzf mruby-build-chroot.img.tar.gz
    mount -o loop mruby-build-chroot.img mruby-build
    sh mruby-build/setup-chroot.sh

The mruby sources are included, update them:

    # we're chrooted now
    cd /root/mruby
    git pull

We need a statically linked mruby binary, so create a new build file:

    mv build_config.rb build_config.rb.bak
    vim build_config.rb

and paste the following content:

```ruby
MRuby::Build.new do |conf|
  conf.cc = ENV['CC'] || 'gcc'
  conf.ld = ENV['LD'] || 'gcc'
  conf.ar = ENV['AR'] || 'ar'
  conf.ldflags = %w(-s -static)
  conf.cflags << (ENV['CFLAGS'] || %w(-g -O3 -Wall -Werror-implicit-function-declaration))
  conf.ldflags << (ENV['LDFLAGS'] || %w(-lm))
end
```

Build mruby:

    make

## Extra fun: copy mruby, mirb, mrbc to the Android filesystem

Exit the chroot

    exit

remount Android's system folder in RW mode:

    mount -o remount -o rw /system
    cp mruby-build/root/mruby/bin/* /system/xbin/

Re-mount /system in RO mode again:

    mount -o remount -o ro /system

Enjoy mruby installed system wide!

```
root@android:/sdcard/ # mirb                                              
mirb - Embeddable Interactive Ruby Shell

This is a very early version, please test and report errors.
Thanks :)

> 
```

## Related links

There're are other (probably much cleaner) ways to do this, like using
the NDK:

[http://podtynnyi.com/2012/11/29/build-mruby-for-android](http://podtynnyi.com/2012/11/29/build-mruby-for-android)

