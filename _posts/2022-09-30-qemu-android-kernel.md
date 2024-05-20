---
layout: post
title:  "Setup Qemu for arm64 android kernel running"
date:   2022-09-30 08:43:59
author: half cup coffee
categories: Linux
tags:	Linux
---

# Setup Qemu for arm64 android kernel running 

## Fetch the android kernel
Google provide the manifest to fetch the kernel soruce, corresponding config files and corss compile toolchain
   
~~~
 mkdir android-kernel && cd android-kernel
 repo init -u https://android.googlesource.com/kernel/manifest -b common-android11-5.4
 repo sync -c -d -j8
~~~
   
Enable the 9P FS for host and guest folder sharing by adding following configs to the kernel config
gki_defconfig is the kernel config file we will used to compile the kernel.
~~~
vi common/arch/arm64/configs/gki_defconfig
 CONFIG_9P_FS_SECURITY=y
 CONFIG_NET_9P=y
 CONFIG_NET_9P_VIRTIO=y
 CONFIG_NET_9P_DEBUG=y
 CONFIG_9P_FS=y
 CONFIG_9P_FS_POSIX_ACL=y
 CONFIG_PCI=y
 CONFIG_VIRTIO_PCI=y
~~~
   
## Compile the kernel
We will combine the rootfs together with kernel imaqge - vmlinux
The configuration CONFIG_INITRAMFS_SOURCE is used to indicate the rootfs place.
~~~
 cd android-kernel/common-modules/virtual-device
 goldfish_defconfig.fragment:CONFIG_INITRAMFS_SOURCE="/media/data-nvm/temp/rootfs/rfs"
 BUILD_CONFIG=common-modules/virtual-device/build.config.goldfish.aarch64  build/build.sh
~~~
   
## Prepare a busybox - the rootfs creation
We will build and generate rootfs from busybox. To build the busybox we need first download a toolchian.
Here is a link for a Linaro toolchain : gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu.tar.xz
It can be downloaded from https://releases.linaro.org/components/toolchain/binaries/latest-5/aarch64-linux-gnu/

For example, we put the toolchain under folder /media/data-nvm/temp/rootfs/gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu,
then run below commands:

~~~
 export PATH=/media/data-nvm/temp/rootfs/gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu/bin:$PATH
 export ARCH=arm64
 export CROSS_COMPILE=aarch64-linux-gnu-
~~~
   
Config the busybix to make it build to static bianry.

~~~
make menuconfig
 setting
 [*] Build static binary (no shared libs)
~~~

Compile busybox:

~~~
make
make install

## below command is important to copy the libc dependencies: 
cp -ar gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/* _install/lib
~~~
   
Create rootfs

~~~
cd _install
mkdir dev etc lib sys proc tmp var home root mnt  
vi etc/profile
 #!/bin/sh
 export HOSTNAME=weller
 export USER=root
 export HOME=/home
 export PS1="[$USER@$HOSTNAME \W]\# "
 PATH=/bin:/sbin:/usr/bin:/usr/sbin
 LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
 export PATH LD_LIBRARY_PATH
~~~


~~~
vi etc/inittab
 ::sysinit:/etc/init.d/rcS
 ::respawn:-/bin/sh
 ::askfirst:-/bin/sh
 ::ctrlaltdel:/bin/umount -a -r
~~~


~~~
vi etc/fstab
 #device  mount-point    type     options   dump   fsck order
 proc /proc proc defaults 0 0
 tmpfs /tmp tmpfs defaults 0 0
 sysfs /sys sysfs defaults 0 0
 tmpfs /dev tmpfs defaults 0 0
 debugfs /sys/kernel/debug debugfs defaults 0 0
 kmod_mount /mnt 9p trans=virtio 0 0
~~~


~~~
vi etc/fstab
 #device  mount-point    type     options   dump   fsck order
 proc /proc proc defaults 0 0
 tmpfs /tmp tmpfs defaults 0 0
 sysfs /sys sysfs defaults 0 0
 tmpfs /dev tmpfs defaults 0 0
 debugfs /sys/kernel/debug debugfs defaults 0 0
 kmod_mount /mnt 9p trans=virtio 0 0
~~~
   
~~~
cd dev/
 sudo mknod console c 5 1
~~~

## Launch with QEMU

sudo apt install qemu-system-arm

~~~
qemu-system-aarch64 -machine virt -cpu cortex-a57 -machine type=virt  -m 1024 -smp 4 -kernel /media/data-sdb2/tmp/android-kernel/out/android11-5.4/dist/Image --append "rdinit=/linuxrc root=/dev/vda rw console=ttyAMA0 loglevel=8"  -nographic  -fsdev local,security_model=passthrough,id=fsdev0,path=//media/data-nvm/temp/rootfs/share -device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=kmod_mount  -drive format=raw,file=/media/data-nvm/temp/rootfs/hd1 -net nic,model=virtio,macaddr=DE:AD:BE:EF:28:05
~~~
   
## boot with disk
~~~
qemu-system-aarch64 -machine virt -cpu cortex-a57 -machine type=virt  -m 1024 -smp 4 -kernel /media/data-sdb2/tmp/android-kernel/out/android11-5.4/dist/Image   -nographic -drive if=none,file=/media/data-nvm/temp/rootfs/rootfs_ext4.img,id=drive-virtio-disk0 -device virtio-blk-device,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1  --append "noinitrd root=/dev/vda rw console=ttyAMA0 loglevel=8"  -fsdev local,security_model=passthrough,id=fsdev0,path=//media/data-nvm/temp/rootfs/share -device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=kmod_mount  -drive format=raw,file=/media/data-nvm/temp/rootfs/hd1 -net nic,model=virtio,macaddr=DE:AD:BE:EF:28:05
~~~
