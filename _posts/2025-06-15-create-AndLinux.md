---
layout: post
title:  "Create a tiny Linux by reusing android build images -- The AndLinux"
subtitle: "WE LIKE THE SIMPLE OF LINUX"
header-img: img/post-bg-coffee.jpeg
date:   2025-06-15 08:00:00
author: half cup coffee
catalog: true
tags:	
    - Linux
    - Android
---

# Introduction

Please refer to this [Re-use the android build images to create a minimal linux system](https://huangweiliang.github.io/2025/01/16/boot-LAGVM-with-busybox/) to get the concept. This page here will
list the steps how to setup such tiny Linux by reusing android build images.

# Fetch the android source code

Yes, we still need android source code. The reason is we still need some tools which come from the android build folder.


# Export PATH for some tools from android source

```java
export ANDROID_ROOT=...
export PATH=$PATH:$ANDROID_ROOT/system/tools/mkbootimg/
export PATH=$PATH:$ANDROID_ROOT/kernel_platform/prebuilts/kernel-build-tools/linux-x86/bin
export PATH=$PATH:$ANDROID_ROOT//out/host/linux-x86/bin
```

# Get the android build images 

Cope the android build images to a folder which you will work with. Those android build images are the image which can make the device work after reflash with them. 

```java
boot.img  super.img  vendor_boot.img
```

# hanlde the super.img

## unsparse a sparse super.img:

```java
simg2img super.img super.img.unsparse
mv super.img.unsparse super_out
```

## use lpunpack to unpack the super.img.unsparse

lpunpack is available after build android in the out folder. In this future this folder can be copied to avoid build android.

```java
$ lpunpack ./super.img.unsparse
Attempting to extract partition 'product_b'...
Attempting to extract partition 'system_ext_b'...
Attempting to extract partition 'product_a'...
  Dealing with extent 0 from target source 0...
Attempting to extract partition 'system_b'...
Attempting to extract partition 'system_a'...
  Dealing with extent 0 from target source 0...
Attempting to extract partition 'vendor_b'...
Attempting to extract partition 'system_ext_a'...
  Dealing with extent 0 from target source 0...
Attempting to extract partition 'system_dlkm_a'...
  Dealing with extent 0 from target source 0...
Attempting to extract partition 'system_dlkm_b'...
Attempting to extract partition 'vendor_a'...
  Dealing with extent 0 from target source 0...
Attempting to extract partition 'vendor_dlkm_b'...
Attempting to extract partition 'vendor_dlkm_a'...
  Dealing with extent 0 from target source 0...

$  ls -l
total 14554372
-rw-r--r-- 1 xx xx  2979463168 Apr 29 13:04 product_a.img
-rw-r--r-- 1 xx xx           0 Apr 29 13:04 product_b.img
-rw-r--r-- 1 xx xx  3783004160 Apr 29 13:04 system_a.img
-rw-r--r-- 1 xx xx           0 Apr 29 13:04 system_b.img
-rw-r--r-- 1 xx xx    12328960 Apr 29 13:04 system_dlkm_a.img
-rw-r--r-- 1 xx xx           0 Apr 29 13:04 system_dlkm_b.img
-rw-r--r-- 1 xx xx   444387328 Apr 29 13:04 system_ext_a.img
-rw-r--r-- 1 xx xx           0 Apr 29 13:04 system_ext_b.img
-rw-r--r-- 1 xx xx   514940928 Apr 29 13:04 vendor_a.img
-rw-r--r-- 1 xx xx           0 Apr 29 13:04 vendor_b.img
-rw-r--r-- 1 xx xx    38473728 Apr 29 13:04 vendor_dlkm_a.img
-rw-r--r-- 1 xx xx           0 Apr 29 13:04 vendor_dlkm_b.img
```
system_dlkm_a.img and vendor_dlkm_a.img are the image files what we need.


# Extract files

## kernel image

```java
$ unpack_bootimg.py --boot_img boot.img --out out_boot
boot magic: ANDROID!
kernel_size: 35289600
ramdisk size: 14241666
os version: 14.0.0
os patch level: 2025-03
boot image header version: 3
command line args:
$ ls -l out_boot/
total 48372
-rw-rw-r-- 1 xx xxh 35289600 Apr 29 13:13 kernel
-rw-rw-r-- 1 xx xxh 14241666 Apr 29 13:13 ramdisk
```

out_boot/kernel is the kernel image.


## device tree image

```java
$ unpack_bootimg.py --boot_img vendor_boot.img --out out_vendor_boot
boot magic: VNDRBOOT
vendor boot image header version: 3
page size: 0x00001000
kernel load address: 0x00008000
ramdisk load address: 0x01000000
vendor ramdisk size: 5037716
vendor command line args: debug user_debug=31 loglevel=0 print-fatal-signals=1 init=/init swiotlb=4096 kpti=0 pcie_ports=compat firmware_class.path=/vendor/firmware_mnt/image loop.max_part=7 androidboot.hardware=qcom androidboot.memcg=1 androidboot.recover_usb=1 androidboot.load_modules_parallel=true androidboot.fstab_suffix=gen3.qcom msm_show_resume_irq.debug_mask=1 log_buf_len=1M console=hvc0,115200 androidboot.console=hvc0 androidboot.selinux=enforcing
kernel tags load address: 0x00000100
product name: 86277384.LU
vendor boot image header size: 2112
dtb size: 1972599
dtb address: 0x0000000001f00000


$ ls -l out_vendor_boot
total 6848
-rw-rw-r-- 1 xx xx 1972599 Apr 29 13:14 dtb
-rw-rw-r-- 1 xx xx 5037716 Apr 29 13:14 vendor_ramdisk
```

The dtb file is the device tree binary.  The vendor_ramdisk is the ramdisk image for android early bootup. we need to get some kernel modules(android first stage modules) from there

## extract first stage kernel modules

The first stage kernel modules are needed to be extract from vendor_ramdisk.

```java
out_vendor_boot$ mkdir out
out_vendor_boot$ cd out
out_vendor_boot/out$ lz4 -dc < ../vendor_ramdisk | cpio -id
31462 blocks

out_vendor_boot/out$ ls -l

first_stage_ramdisk
lib

out_vendor_boot/out$ ls -l lib/modules/

aquantia.ko                     mhi.ko                          pinctrl-monaco_auto.ko          smcinvoke_dlkm.ko
arm_smmu.ko                     minidump.ko                     pinctrl-msm.ko                  smem.ko
bootmarker_proxy.ko             modules.alias                   pinctrl-sdmshrike.ko            socinfo_dt.ko
boot_stats.ko                   modules.blocklist               pinctrl-slpi.ko                 spidev.ko
cfg80211.ko                     modules.dep                     pinctrl-sm6150.ko               spi-msm-geni.ko
clk-dummy.ko                    modules.load                    pinctrl-sm8150.ko               stmmac.ko
........
```

This kernel module folder need to be copied to the busybox rootfs image. Busybox rootfs image is the image be loaded by kernel. So the first stage modules need to be loaded from there because these are basic drivers suck as virtio-blk.
We will talk about more later about image creations. 



## extract kernel modules from vendor_dlkm

Back to the super_out folder:

```java
super_out$ sudo mount -o loop,ro vendor_dlkm_a.img for_mount/
super_out$ ls for_mount/

etc  lib  lost+found

cd for_mount/lib/modules

super_out/for_mount/lib/modules$ ls
adsp_loader_dlkm_legacy.ko  heap_mem_ext_v01.ko             msm-vm-poweroff.ko              q6_notifier_dlkm_legacy.ko  snd_event_dlkm_legacy.ko
apr_dlkm.ko                 hsi2s.ko                        native_dlkm.ko                  qca_cld3_qca6390.ko         socinfo_dt.ko
...

```

There is a file modules.load in this folder.  This file will be used as a configuration file for a script to load the modules. 
Modules' sequence must follow the definition from this file.
This kernel module folder(lib/modules) need to be copied to the data.img (vendor_dlkm). This data.img is a block device be pased to the tinylinux then we can load these modules by script.
We will talk about more later about image creations. 


## Extract kernel modules from system_dlkm

Back to the super_out folder:

```java
super_out$ sudo mount -o loop,ro system_dlkm_a.img for_mount/
```

The modules folder structure is different from the vendor_dlkm.

```java
super_out$ ls for_mount/lib/modules/6.1.99-android14-11-maybe-dirty/ -l

total 428
drwxr-xr-x. 8 root root   4096 Apr 15 15:54 kernel
-rw-r--r--. 1 root root  67078 Apr 15 15:54 modules.alias
-rw-r--r--. 1 root root  69016 Apr 15 15:54 modules.alias.bin
-rw-r--r--. 1 root root  22284 Apr 15 15:54 modules.builtin
-rw-r--r--. 1 root root  25913 Apr 15 15:54 modules.builtin.alias.bin
-rw-r--r--. 1 root root  26267 Apr 15 15:54 modules.builtin.bin
-rw-r--r--. 1 root root 141192 Apr 15 15:54 modules.builtin.modinfo
-rw-r--r--. 1 root root   4229 Apr 15 15:54 modules.dep
-rw-r--r--. 1 root root   6846 Apr 15 15:54 modules.dep.bin
-rw-r--r--. 1 root root     97 Apr 15 15:54 modules.devname
-rw-r--r--. 1 root root   1955 Apr 15 15:54 modules.load
-rw-r--r--. 1 root root   1955 Apr 15 15:54 modules.order
-rw-r--r--. 1 root root     55 Apr 15 15:54 modules.softdep
-rw-r--r--. 1 root root  14590 Apr 15 15:54 modules.symbols
-rw-r--r--. 1 root root  18117 Apr 15 15:54 modules.symbols.bin
```

There is a file modules.load in this folder.  This file will be used as a configuration file for a script to load the modules. 
Modules' sequence must followe the definition from this file.

This kernel module folder(lib/modules) need to be copied to the data.img (system_dlkm). This data.img is a block device be passed to the linux guest then we can load these modules by script.
We will talk about more later about image creations. 

# Create images

## Busybox root filesystem image
How to compile the busybox and create a rootfs from scratch will not include in this section. Because it is not necessary to create everytime from scratch.
Here we only talk about to work with a available busybox image.  To add configuration, scripts and kernel modules and generate a updated image.

Introduction to the busybox image:

```java
sudo mount -o loop,rw bxrfs.img for_mount/
```

The file for_mount/etc/init.d/rcS is the init script when this image it loaded:


```java
for_mount/etc/init.d$ cat rcS
mkdir -p /sys
mkdir -p /tmp
mkdir -p /proc
mkdir -p /mnt
mkdir -p /mnt/data
mkdir -p /mnt/vda
mkdir -p /vendor/
mkdir -p /system
mkdir -p /vendor/firmware_mnt   ## this is the partition for FW. we will mount the corresponding partition here.

ln -s /mnt/data/vendor_dlkm/lib/ /vendor/  ## /mnt/data/ is the folder which mounted with data.img. Here create a folder linked to the vendor_dlkm modules
ln -s /mnt/data/system_dlkm/lib/ /system   ## /mnt/data/ is the folder which mounted with data.img. Here create a folder linked to the system_dlkm modules

/bin/mount -a
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
mdev -d

echo "===============load modules==============="
/bin/sh /etc/load_boot_m.sh     ## this is the script to load the first stage modules


/bin/mount -a   ## mount all partitions defined in the for_mount/etc/fstab
```

The content of for_mount/etc/fstab:

```java
#device  mount-point    type     options   dump   fsck order
proc /proc proc defaults 0 0
tmpfs /tmp tmpfs defaults 0 0
sysfs /sys sysfs defaults 0 0
tmpfs /dev tmpfs defaults 0 0
debugfs /sys/kernel/debug debugfs defaults 0 0
/dev/vda /mnt/vda  ext4  defaults 0 0
/dev/vdb /mnt/data  ext2  defaults 0 0
/dev/vde /vendor/firmware_mnt vfat defaults 0 0
```
The folder for_mount/lib/modules is the place to store the first stage modules.
Delete for_mount/lib/modules and replace with the folder from the new first stage kernel modules you extracted.

Then you get a updated busybox image.

## 6.2 data image for the rest of kernel modules

data.img is a 512MB image only stores the kernel modules extracted from vendor_dlkm and system_dlkm image.
we don't need create data.img from scrach. Just update current available one is enough.

```java
sudo mount -o loop,ro data.img for_mount/

ls for_mount/
lost+found  system_dlkm  vendor_dlkm
```
system_dlkm and vendor_dlkm folder are contains the lib/modules from system_dlkm.img and vendor_dlkm.img.

# 7. Introduction for the QVM config

Because we already have the qvm configuration for android bootup. We can recreate a QVM config based on that. Below provide the delta difference:

```java

load /var/test_folder/kernel  # Load the kernel
initrd load /var/test_folder/bxrfs.img  # Load the fs image as ramdisk
fdt load /var/test_folder/sa8155-vm-xxx.dtb # Load the device tree blob

# The kernel command line options
cmdline "console=ttyAMA0 earlycon=pl011,0x1c090000 rw rootfstype=ext2 root=/dev/ram0 init=/linuxrc ramdisk_size=65536 swiotlb=4096 kpti=0 pcie_ports=com    pat firmware_class.path=/vendor/firmware_mnt/image loop.max_part=7"

# UART console
vdev vdev-pl011.so loc 0x1c090000 intr gic:37 sched 20 hostdev /dev/ptyqa

# VDA for debian rootfs,  it is not used.
vdev vdev-virtio-blk.so sched 10 loc 0x1c0b0000  intr gic:40 threads 4 hostdev  /var/test_folder/debian.img
# VDB for store vendor_dlkm and system_dlkm kernel modules 
vdev vdev-virtio-blk.so sched 10 loc 0x1c0e0000 intr gic:39 threads 4 hostdev /var/test_folder/data.img

# VDE the android original partition which includes the FW images
vdev vdev-virtio-blk.so sched 10 loc 0x1c110000  intr gic:44 threads 4 hostdev /dev/disk/modem_a dio enable


```

# 8. Example log to update the images from android images

```java
sudo mount -o loop,ro system_dlkm_a.img for_mount_system_dlkm
sudo mount -o loop,ro vendor_dlkm_a.img for_mount_vendor_dlkm

sudo mount -o loop,rw data.img for_mount
cp ../../../super_out/for_mount_system_dlkm/lib/ ./ -a

for_mount$ tree -L 4 system_dlkm/
system_dlkm/
└── lib
    └── modules
        └── 6.1.99-android14-11-maybe-dirty
            ├── kernel
            ├── modules.alias
            ├── modules.alias.bin
            ├── modules.builtin
            ├── modules.builtin.alias.bin
            ├── modules.builtin.bin
            ├── modules.builtin.modinfo
            ├── modules.dep
            ├── modules.dep.bin
            ├── modules.devname
            ├── modules.load
            ├── modules.order
            ├── modules.softdep
            ├── modules.symbols
            └── modules.symbols.bin

we meed to move the folder lib/modules/6.1.99-android14-11-maybe-dirty/* to the upper layer folder with command like :
```
mv system_dlkm/lib/modules/6.1.99-android14-11-maybe-dirty/*  system_dlkm/lib/modules/
```java

for_mount/vendor_dlkm$ tree -L 2 lib/
lib/
└── modules
    ├── adsp_loader_dlkm_legacy.ko
    ├── apr_dlkm.ko
    ├── aquantia.ko
    ├── boot_marker.ko
    ├── btpower.ko
    ├── cfg80211.ko
    ├── clk-dummy.ko
    ├── clk-qcom.ko
    ├── cnss2.ko
    ├── cnss_nl.ko
......

cd sudo mount -o loop,rw bxrfs.img for_mount/
sudo umount for_mount/


sudo mount -o loop,rw bxrfs.img for_mount/
cp -ar ../../../out_vendor_boot/out/lib/modules/ ./

cd sudo mount -o loop,rw bxrfs.img for_mount/
sudo umount for_mount/
```

Copy files to target and launch it: Below are the files you need to copy to

```java

# ls -l /var/00000000/
total 2376927
-rwxrwxrwx   1 root      root       67108864 May 06 12:13 bxrfs.img
-rwxrwxrwx   1 root      root      536870912 May 04 23:24 data.img
-rwxrwxrwx   1 root      root      536870912 May 04 23:23 debian.img
-rwxrwxrwx   1 root      root          12518 May 04 23:19 high_qvm_config.config
-rw-rw-r--   1 root      root       35289600 May 05 09:46 kernel
-rwxrwxrwx   1 root      root           3494 May 05 09:53 vdisk.config
```

Stop current qvm:
```java
dtach -a /tmp/vmm
su
reboot -p

```

wait 10s to launch the tiny Linux

```java
export GUESTDUMP_ENABLE=0
qvm @/var/00000000/high_qvm_config.config  @/var/00000000/vdisk.config &
dtach -a /tmp/vmm

```
After boot up, we need execute 3 scripts to load the modules:

```java
sh system_dlkm_modprobe.sh

sh set_wlan_fw.sh
-rwxr--r--    1 70675579 70675579     12786 Jun  6  2025 /mnt/data/vendor/firmware/wlan/qca_cld/qca6390/WCNSS_qcom_cfg.ini
total 16
-rwxr--r--    1 0        0            12786 Jan  1 00:04 WCNSS_qcom_cfg.ini

sh vendor_modprobe.sh

lspci

    01:00.0 Class 0280: 17cb:1101
    00:00.0 Class 0604: 17cb:0109


ifconfig wlan0
    wlan0     Link encap:Ethernet  HWaddr E0:2D:F0:98:75:54
              BROADCAST MULTICAST  MTU:1500  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:3000
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

ifconfig wlan1
    wlan1     Link encap:Ethernet  HWaddr E0:2D:F0:19:75:54
              BROADCAST MULTICAST  MTU:1500  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:3000
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)


```

# Till now, the Linux and kernel driver modules are all loadded. 


# Prepare a busybox - the rootfs creation
We will build and generate rootfs from busybox(here with busybox-1.35.0.tar.bz2). To build the busybox we need first download a toolchian.
Here is a link for a Linaro toolchain : gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu.tar.xz
It can be downloaded from https://releases.linaro.org/components/toolchain/binaries/latest-5/aarch64-linux-gnu/

For example, we put the toolchain under folder /media/data-nvm/temp/rootfs/gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu,
then run below commands:

```java
 export PATH=/...../gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu/bin:$PATH
 export ARCH=arm64
 export CROSS_COMPILE=aarch64-linux-gnu-
```
   
Config the busybix to make it build to static bianry.

```java
make menuconfig
 setting
 [*] Build static binary (no shared libs)
```

Compile busybox:

```java
make
make install

## below command is important to copy the libc dependencies: 
cp -ar gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/* _install/lib
```
   
Create rootfs

```java
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
```


```java
vi etc/inittab
 ::sysinit:/etc/init.d/rcS
 ::respawn:-/bin/sh
 ::askfirst:-/bin/sh
 ::ctrlaltdel:/bin/umount -a -r
```


```java
vi etc/fstab
 #device  mount-point    type     options   dump   fsck order
 proc /proc proc defaults 0 0
 tmpfs /tmp tmpfs defaults 0 0
 sysfs /sys sysfs defaults 0 0
 tmpfs /dev tmpfs defaults 0 0
 debugfs /sys/kernel/debug debugfs defaults 0 0
 kmod_mount /mnt 9p trans=virtio 0 0
```


```java
vi etc/fstab
 #device  mount-point    type     options   dump   fsck order
 proc /proc proc defaults 0 0
 tmpfs /tmp tmpfs defaults 0 0
 sysfs /sys sysfs defaults 0 0
 tmpfs /dev tmpfs defaults 0 0
 debugfs /sys/kernel/debug debugfs defaults 0 0
 kmod_mount /mnt 9p trans=virtio 0 0
```
   
```java
cd dev/
 sudo mknod console c 5 1
```