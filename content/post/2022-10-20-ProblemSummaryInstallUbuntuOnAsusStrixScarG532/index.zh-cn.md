---
title: 在ROG枪神4上安装Ubuntu遇到的问题和解决办法
description: 在ROG枪神4上安装Ubuntu遇到的问题总结
date: 2022-10-20 13:22:01+0800
categories:
    - Linux
    - Solution
tags:
    - Ubuntu
    - Linux
image: https://cdn.staticaly.com/gh/Misakaou/imagestorage@master/20221020/1880882405-install-Problem,-Ubuntu,-webpage-head-image.g4gmdlq05jc.webp
---

本文来源: [MoeomuBlog](/zh-cn/posts/在rog枪神4上安装ubuntu遇到的问题和解决办法/)

## 声音的问题

### 问题一：没有声音

1. 编辑文件：`/etc/modprobe.d/alsa-base.conf`
2. 在这个文件的末尾添加一行：`options snd-hda-intel model=asus-zenbook`
3. 重新启动系统

### 问题二：声音无法调整

> 描述：主声音不可调整，只能是静音或者最大声音。

- 编辑文件：`/usr/share/pulseaudio/alsa-mixer/paths/analog-output.conf.common`
- 在`[Element PCM]`的前面添加以下内容

  ```text
  [Element Master]
  switch = mute
  volume = ignore
  ```

- 关闭pulse音频守护进程，它将被systemd自动重新启动：`pulseaudio -k`

> 注意：请勿以root身份运行pulseaudio，否则将无法正常关闭守护进程。

## 睡眠问题

### 锁定屏幕后无法再次唤醒

> 描述：锁定屏幕后，计算机不能再次唤醒。硬盘和其他硬件工作正常，但屏幕全黑。

1. 安装laptop-mod-tools：`sudo apt install laptop-mod-tools`
2. 启动laptop-mod-tools：`sudo laptop_mode start`
3. 添加mutter debug环境变量，编辑文件：`/etc/environment`
   - 如果是Ubuntu 22.04: `MUTTER_DEBUG_ENABLE_ATOMIC_KMS=0`
   - 如果是Ubuntu 22.10及以后版本：`MUTTER_DEBUG_FORCE_KMS_MODE=simple`
4. 重新启动系统

## 参考文献

- [snd-hda-intel model for UBUNTU 20.04 installed in an ASUS ROG STRIX G15 (G512) - AskUbuntu](https://askubuntu.com/questions/1288054/snd-hda-intel-model-for-ubuntu-20-04-installed-in-an-asus-rog-strix-g15-g512)
- [Volume either muted or maxed out, nothing in between - ubuntu forums](https://ubuntuforums.org/showthread.php?t=2414755)
- [Blanked screen doesn't wake up after locking - Launchpad](https://bugs.launchpad.net/ubuntu/+source/mutter/+bug/1968040)
