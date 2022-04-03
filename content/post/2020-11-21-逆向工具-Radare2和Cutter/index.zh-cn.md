---
title: 逆向工具-Radare2和Cutter
description: 开源逆向神器-Radare2和Cutter
date: 2020-11-21 19:04:00+0800
categories:
    - Reverse
tags:
    - Reverse
    - Tools
---

本文来源：[Moeomu的博客](/zh-cn/posts/逆向工具-radare2和cutter/)

## 介绍

### Radare2

- r2 is a rewrite from scratch of radare in order to provide a set of libraries and tools to work with binary files.

- Radare project started as a forensics tool, a scriptable command-line hexadecimal editor able to open disk files, but later added support for analyzing binaries, disassembling code, debugging programs, attaching to remote gdb servers...

- radare2 is portable.

- To learn more you may read the official radare2 book, the source code, or browse the web for blog posts or presentations from r2con.

### Cutter

- Cutter is a free and open-source reverse engineering framework powered by radare2 . Its goal is making an advanced, customizable and FOSS reverse-engineering platform while keeping the user experience at mind. Cutter is created by reverse engineers for reverse engineers.

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
