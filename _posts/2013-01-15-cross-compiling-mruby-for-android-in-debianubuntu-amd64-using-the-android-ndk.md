---
layout: post
title: "Cross-compiling mruby for Android in Debian/Ubuntu amd64 using the Android NDK"
description: ""
category: hack
tags: [ubuntu, mruby, ruby, android]
---
{% include JB/setup %}

Following the previous [Building mruby-head with your Android Phone](/hack/2013/01/11/building-mruby-with-your-android-phone), here's the step by step guide to create a mruby binary for your Android phone.

Cleaner approach that does not need static linking and works without having to
install a chroot in your phone.

## 1. Install i386 packages

Some binaries in the NDK are not 64 binaries, so these packages are required to have 
a fully functional Android NDK on amd64:

    sudo apt-get install gcc-multilib

## 2. Download and install the Android NDK

Get it from [developer.android.com](https://developer.android.com/tools/sdk/ndk/index.html), Linux 32/64-bit (x86) edition.

Once you have it, create a directory for it and unpack it.

Assuming the NDK tarball has been downloaded to **$HOME/downloads**
and it's the **r8d** release:

    mkdir ~/android/ -p
    cd ~/android
    tar xjf ~/downloads/android-ndk-r8d-linux-x86.tar.bz2

Create a standalone toolchain:

    # API Level 14, Android 4.0 and above
    ~/android/android-ndk-r8d/build/tools/make-standalone-toolchain.sh \
                                         --platform=android-14 \
                                         --install-dir=$HOME/android/toolchain-p14

Create a script to export some required environment variables later on:

    vim ~/android/env-p14.sh

Open it with your favorite text editor and paste the following content:

```bash
#!/bin/sh
export NDK_ROOT=$HOME/android/toolchain-p14
export PATH="$NDK_ROOT/bin:$PATH"
export SYS_ROOT="$NDK_ROOT/sysroot"
export CFLAGS="--sysroot=$SYS_ROOT"
export CC="arm-linux-androideabi-gcc"
export LD="arm-linux-androideabi-gcc"
#export LDFLAGS=
export AR="arm-linux-androideabi-ar"
export RANLIB="arm-linux-androideabi-ranlib"
export STRIP="arm-linux-androideabi-strip"
export MRBC_BIN=$HOME/android/work/mruby/xcompile/mrbc
```

## 3. Download mruby

    mkdir ~/android/work/ && cd ~/android/work
    git clone https://github.com/mruby/mruby

## 4. Build mruby (for your current arch)

We'll need amd64 mruby binaries to build for ARM, so build mruby first:

    cd ~/android/work/mruby
    make
   
## 5. Save mruby (i386) binaries

Wait for the build to finish and copy the resulting binaries to
~/android/work/mruby/xcompile/ (you'll need to adjust the MRBC_BIN in the env.sh
script created in step #1 if you change this path): 

    mkdir ~/android/work/mruby/xcompile
    cp ~/android/work/mruby/bin/{mruby,mrbc,mirb} ~/android/work/mruby/xcompile/ 
    # clean the build after that
    make clean

## 6. Setup the xcompile environment

Source the env.sh script we created in step #1 to export the required ENV variables:

    source ~/android/env-p14.sh

We're now using ARM GCC at this point.

## 7. Patch mruby build system

Create **$HOME/android/work/mruby/xcompile/mruby.diff** file and paste the following contents:

```diff
diff --git a/tasks/mruby_build.rake b/tasks/mruby_build.rake
index de9e556..0750eb6 100644
--- a/tasks/mruby_build.rake
+++ b/tasks/mruby_build.rake
@@ -101,7 +101,7 @@ module MRuby
     end
 
     def mrbcfile
-      @mrbcfile ||= exefile("build/host/bin/mrbc")
+      @mrbcfile ||= (ENV['MRBC_BIN'] || exefile("build/host/bin/mrbc"))
     end
 
     def compile_c(outfile, infile, flags=[], includes=[])

```

Patch the sources:

    cd ~/android/work/mruby
    patch -p1 < xcompile/mruby.diff

## 8. Build the sources for ARM

We're ready to cross-compile the sources:

    make

## 9. Profit

```
[9602][rubiojr.blueleaf] file bin/mruby 
bin/mruby: ELF 32-bit LSB executable, ARM, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
```

mruby,mirb and mrbc Android (ARM) binaries are now available in the bin/ directory. 
Copy them to your Android device and enjoy mruby!
