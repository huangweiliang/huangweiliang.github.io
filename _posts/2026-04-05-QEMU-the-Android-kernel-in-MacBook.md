---
layout: post
title:  "Setup Linux kernel development env with macbook"
subtitle: "play with new macbook"
header-img: img/post-bg-coffee.jpeg
date:   2026-04-05 08:55:59
author: half cup coffee
catalog: true
tags:	
    - Android
    - Linux
---

Recently I switched from windows to Macbook which is my first time to use the MacOS. It is
fresh to me like the feeling when I start to use Linux OS. Well, I knew MacOS is unix-like
OS, so it has the genie like Linux. I bought the __Macbook M5 Pro__ with 2TB SSD and 48GB RAM.
My intension is to use it to learn AI stuff, also I do need a personal laptop for my master
studying. I have a very old laptop which is over 10 years. It is also the time to buy a new
laptop. Although most of time I am using company's laptop to process some personal topics.

I am thinking how to use it to do potential development and investigation. The major requirement
from me is about Linux and QNX development and study. For Linux development, my major purpose
is about kernel compiling and simulation by the qemu. I had some introduction before[[1]](https://huangweiliang.github.io/2024/07/17/qemu_linux_61/) [[2]](https://huangweiliang.github.io/2022/09/30/qemu-android-kernel/) to setup
the qemu environment for the [google android kernel project](https://source.android.com/docs/setup/build/building-kernels), combine with a busybox I can lunch
a tiny linux with android kernel, which makes me possible to debug and study the android kernel.
Google android kernel project is not just kernel, it also includes the complete toolchain and
build environment such as Bazel. Unfortunately, Bazel is not supported well by MacOS. I have to
install virtual machine. The best virtial machin is [Orbstack](https://orbstack.dev/) in MacOS. I tried Docker but the 
corss compile efficiency is worse than Orbstack. As you know the biggest problem is M serials
apple silicon is ARM based processor, we need a VM with better efficiency for the x86/amd64
simulation. Although the Orbstack still slower than the native Linux machine, but it is 
acceptable.  I tried google project android14-6.1 branch. It can be built successfully.

But the android16-6.12 branch can not be built. I see there is a ticket reported to Orbstack some 
months ago. The issue is not sovled yet. So to build the android kernel v6.12 I tried to build
it manually which w/o the Bazel environment.

Fetch the code:

```bash

git clone --branch android16-6.12-lts --single-branch --depth 1 https://android.googlesource.com/kernel/common

```

Setup build env:
we can reuse the prebuilts folder from any branch of google android kernel project.

```bash
export PATH=$PATH:/Volumes/my1tb/test/prebuilts/clang/host/linux-x86/clang-r536225/bin/
export ARCH=arm64
export SUBARCH=arm64
export CLANG_TRIPLE=aarch64-linux-gnu-
```

Generate kernel configuration:

```bash
make O=out LLVM=1 gki_defconfig
```

Please note that after the .config is generated, we need to modify the out/.config to add any
drivers we need and the initramfs to compile the busybox ramdisk into the kernel image.

Compile the kernel:
```bash
make O=out LLVM=1 -j4
```

Generate the Image:

```bash
llvm-objcopy -O binary -R .notes -R .comment -S vmlinux Image
```

Launch the Kernel with QEMU: simplest command

```bash
qemu-system-aarch64 -machine virt -cpu cortex-a57 -machine type=virt  -m 1024 -smp 4 -kernel /Users/huang/workspace/android/Image2  --append "rdinit=/linuxrc root=/dev/vda rw console=ttyAMA0 loglevel=8"  -nographic  -accel hvf
```







