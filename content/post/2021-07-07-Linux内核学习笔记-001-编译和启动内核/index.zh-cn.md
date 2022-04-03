---
title: Linux内核学习笔记-001-编译和启动内核
description: Linux内核学习笔记-001-编译和启动内核
slug: linux-kernel-study-001
date: 2021-07-07 11:20:00+0800
categories:
    - Kernel
tags:
    - Linux
    - Kernel
---

本文来源：[Moeomu的博客](/p/linux-kernel-study-001/)

## 准备环境

- Ubuntu 20.04 LTS
- Linux 4.9.229
- Busybox 1.33.0
- qemu

### 下载内核源代码和文件系统源代码

- 在站点[Kernel.org](https://www.kernel.org)下载`linux-4.9.229.tar.gz`
- 在站点[Busybox.net](https://www.busybox.net/downloads)下载`busybox-1.33.0.tar.bz2`
- 通过apt安装qemu：`sudo apt install qemu-system-x86`

## 编译

### 编译内核

- `export ARCH=x86`
- `make x86_64_defconfig`
- `make menuconfig`
  - 选中`General Setup -> Initial RAM filesystem and RAM disk(initramfs/initrd) support`
  - 选中`Device Drivers -> Block devices -> RAM block device support`
  - 修改`Device Drivers -> Block devices -> RAM block device support -> (65536)default RAM disk size (kbytes)`

> 如果这一步报错`fatal error: curses.h`，则安装一下`sudo apt install libncurses5-dev`

- `make`

- 编译好的内核被放置在目录`arch/x86/boot`下，文件名`bzImage`

### 编译busybox

- `make menuconfig`
  - 选中`Settings -> Build Options -> [*] Build Busybox as a static binary (no shard libs)`
- `make && make install`

## 打包文件系统

- `mkdir etc dev mnt`
- `mkdir -p proc sys tmp`
- `mkdir -p etc/init.d`
- `vim etc/fstab`
  
  ```text
  proc /proc proc defaults 0 0
  tmpfs /tmp tmpfs defaults 0 0
  sysfs /sys sysfs defaults 0 0
  ```

- `vim etc/init.d/rcS`

  ```shell
  echo -e "Welcome to Moeomu Linux"
  /bin/mount -a
  echo -e "Remounting the root filesystem"
  mount -o remount rw /
  mkdir -p /dev/pts
  moutn -t devpts devpts /dev/pts
  echo /sbin/mdev > /proc/sys/kernel/hotplug
  mdev -s
  ```

- `chmod 755 etc/init.d/rcS`
- `vim etc/inittab`
  
  ```text
  ::sysinit:/etc/init.d/rcS
  ::respawn:-/bin/sh
  ::askfirst:-/bin/sh
  ::cttlaltdel:/bin/umount -a -r
  ```

- `chmod 755 etc/inittab`
- `cd dev`
- `sudo mknod console c 5 1`
- `sudo mknod null c 1 3`
- `sudo mknod tty1 c 4 1`

- 以下代码在busybox源码目录下逐行执行
  
  ```shell
  #!/bin/bash
  rm -rf rootfs.ext3
  sudo rm -rf fs
  dd if=/dev/zero of=./rootfs.ext3 bs=1M count=32
  mkfs.ext3 rootfs.ext3
  mkdir fs
  sudo mount -o loop rootfs.ext3 ./fs
  sudo cp -rf ./_install/* ./fs
  sudo umount ./fs
  gzip --best -c rootfs.ext3 > rootfs.img.gz

  ```

- 最终生成了文件系统：`rootfs.img.gz`

## 使用QEMU运行系统

- `qemu-system-x86_64 -kernel ./linux-4.9.229/arch/x86_64/boot/bzImage -initrd ./busybox-1.33.1/rootfs.img.gz -append "root=/dev/ram init=/linuxrc" -serial file:output.txt`

- 预览如下

  ![RHildI.png](https://z3.ax1x.com/2021/07/07/RHildI.png)
