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
```

## 3. Download mruby

    mkdir ~/android/work/ && cd ~/android/work
    git clone https://github.com/mruby/mruby

## 4. Create a new build file

Create a new build_config.rb file in ~/android/work/mruby:

    rm ~/android/work/mruby/build_config.rb
    vim ~/android/work/mruby/build_config.rb

and paste the following contents:

```ruby
MRuby::Build.new do |conf|
  conf.cc = ENV['CC'] || 'gcc'
  conf.ld = ENV['LD'] || 'gcc'
  conf.ar = ENV['AR'] || 'ar'
  conf.cflags << (ENV['CFLAGS'] || %w(-g -O3 -Wall -Werror-implicit-function-declaration))
  conf.ldflags << (ENV['LDFLAGS'] || %w(-lm))
end

MRuby::CrossBuild.new('arm') do |conf|
  conf.cc = ENV['CC'] || 'arm-linux-androideabi-gcc'
  conf.ld = ENV['LD'] || 'arm-linux-androideabi-gcc'
  conf.ar = ENV['AR'] || 'arm-linux-androideabi-ar'
  conf.cflags << (ENV['CFLAGS'] || %w(-g -O3 -Wall -Werror-implicit-function-declaration))
end

```

And source the env-p14.sh script so the build finds the toolchain:

    source ~/android/env-p14.sh

## 5. Build mruby

    cd ~/android/work/mruby
    make
   
## 6. Profit

```
[9602][rubiojr.blueleaf] file build/arm/bin/{mruby,mirb,mrbc}
build/arm/bin/mruby: ELF 32-bit LSB executable, ARM, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
build/arm/bin/mirb:  ELF 32-bit LSB executable, ARM, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
build/arm/bin/mrbc:  ELF 32-bit LSB executable, ARM, version 1 (SYSV), dynamically linked (uses shared libs), not stripped
```

mruby,mirb and mrbc Android (ARM) binaries are now available in the **build/arm/bin** directory. 
Copy them to your Android device and enjoy mruby!
