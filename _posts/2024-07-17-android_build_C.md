---
layout: post
title:  "Manually cross compile a program for android"
subtitle: "Get rid of the android build system"
header-img: img/post-bg-coffee.jpeg
date:   2024-07-17 09:00:00
author: half cup coffee
catalog: true
tags:	
    - Android
---


# Cross compile a programm for android

Android build system will compole all the software components includes the kernel, native libraries and so on. Normally if we are going to add a native program we need follow the android rules to add the makefile Android.bp or Android.mk. If the program only consist of several source files or even single file. It is easy for us to add the android makefile for it. But sometimes we want to port opensource program to android which it might include hundreds of source files and many sub folders, also need feature configuration before we compile it.

Normally for a cross compile we just need specify the cross compiler and the sysroot then we can build a program. We want the compile for android can be as simple as this way. So here give a guidence for that purpose:

```
export TOOLCHAIN=/PROJECT_ROOT/prebuilts/ndk-r23/toolchains/llvm/prebuilt/linux-x86_64/
export TARGET=aarch64-linux-android
export API=23
export AR=$TOOLCHAIN/bin/llvm-ar
export CC=$TOOLCHAIN/bin/$TARGET$API-clang
export AS=$CC
export CXX=$TOOLCHAIN/bin/$TARGET$API-clang++
export LD=$TOOLCHAIN/bin/ld
export RANLIB=$TOOLCHAIN/bin/llvm-ranlib
export STRIP=$TOOLCHAIN/bin/llvm-strip

##In case there is autogen.sh:
./autogen.sh --enable-tools=yes --host=$TARGET --with-sysroot=$TOOLCHAIN/sysroot --prefix=xxxxx

##In case there is configure:
./configure --host=$TARGET --with-sysroot=$TOOLCHAIN/sysroot --prefix=xxxxx

make 
make install
```
