---
title: 逆向工具-Radare2和Cutter
description: 开源逆向神器-Radare2和Cutter
date: 2020-11-21 19:04:00+0800
categories:
    - 软件推荐
tags:
    - Reverse
    - Tools
---

本文来源：[Moeomu的博客](/zh-cn/posts/逆向工具-radare2和cutter/)

## 介绍

### Radare2

- r2是对radare的一次重写，目的是提供一套处理二进制文件的库和工具。

- Radare项目开始时是一个取证工具，一个能够打开磁盘文件的可编写的命令行十六进制编辑器，但后来增加了对分析二进制文件、反汇编代码、调试程序、附加到远程gdb服务器的支持。

- radare2是可移植的。

- 要了解更多信息，你可以阅读官方的radare2书籍，源代码，或者浏览网络上的博客文章或r2con的演讲。

### Cutter

- Cutter是一个由radare2驱动的免费和开源的逆向工程框架。它的目标是建立一个先进的、可定制的、FOSS的逆向工程平台，同时考虑到用户体验。Cutter是由逆向工程师为逆向工程师创建的。

## 安装

> 由于Cutter是Radare2的GUI化程序，因此只需要下载Cutter即可

- 本次测试系统：macOS Catalina 10.15.7
- Github下载链接：[github.com/radareorg/cutter/releases](https://github.com/radareorg/cutter/releases)
- Moeomu网盘下载链接(可能不是最新)：[Cutter-v1.12.0-x64.dmg](https://pan.moeomu.com/Software/macOS/Tools-Reverse_Pwn/Cutter-v1.12.0-x64.macOS.dmg)

## macOS Catalina无法运行Cutter的原因

> 不知道Windows怎么样，反正在macOS Catalina上安装后无法正常运行，解决方法如下

- 找出无法正常运行的原因
- 进入Cutter的文件夹：`cd /Applications/Cutter.app/Contents/MacOS/`
![CD-Cutter-Folder](https://s3.ax1x.com/2020/11/21/D3Sgds.png)
- 直接运行Cutter查看原因：`./Cutter`
![Cutter-Error-Message](https://s3.ax1x.com/2020/11/21/D3S2on.png)
- 安装`gettext`解决问题：`brew install gettext`
![install-gettext](https://s3.ax1x.com/2020/11/21/D3SWiq.png)
