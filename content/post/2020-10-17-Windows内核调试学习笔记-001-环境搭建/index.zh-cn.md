---
title: Windows内核调试学习笔记-001-环境搭建
description: Windows内核的相关的一些学习笔记，这一篇主要是环境的搭建
#slug: windows-kernel-debug-001
date: 2020-10-17 19:27:00+0800
#image: cover.jpg
categories:
    - Kernel
tags:
    - Windows
    - Kernel
---

本文来源：[Moeomu的博客](/zh-cn/posts/Windows内核调试学习笔记-001-环境搭建/)

## 下载工具

- Windows 7 SP1 x86 [镜像迅雷下载链接](thunder://QUFlZDJrOi8vfGZpbGV8Y25fd2luZG93c183X3VsdGltYXRlX3dpdGhfc3AxX3g4Nl9kdmRfdV82Nzc0ODYuaXNvfDI2NTMyNzYxNjB8NzUwM0U0QjlCODczOERGQ0I5NTg3MjQ0NUM3MkFFRkJ8L1pa)
- VMWare Workstation 16(链接在下方)
- WinDbg Preview([Microsoft Store](https://www.microsoft.com/zh-cn/p/windbg-preview/9pgjgd53tn86))

---

## 安装Windows虚拟机

> 最初以Windows 7 SP1 x86为例子来学习

- MSDN下载官方镜像
- VMWare Workstation 16搭建虚拟环境
  - 下载：[VMWare 16 Link](https://www.vmware.com/go/getworkstation-win)
  - 密钥：`ZF3R0-FHED2-M80TY-8QYGC-NPKYF`

---

## 配置Windows内核调试虚拟机

### 移除此虚拟机的打印机设备

### 添加串行串口

- 点击使用命名的管道
- 填入字符串：`\\.\pipe\Windows7x86`(可以填入自己希望的管道命名，但是只能修改`Windows7x86`位置处)
- 下方选择该端是服务器，另一端是应用程序
- I/O模式中，选择`轮询时主动放弃`

> 配置完成如下图所示

![虚拟机配置图.png](https://s1.ax1x.com/2020/10/18/0j8UKI.png)

### 配置Windows7

- 输入命令`msconfig`，点击引导，如图

![引导.jpg](https://s1.ax1x.com/2020/10/18/0jGpJe.png)
  
- 点击高级选项，启用调试，波特率，如图

![高级选项.jpg](https://s1.ax1x.com/2020/10/18/0jGSiD.png)

---

## 配置WinDbg Preview

- 首先启动代理网络，用于解除GFW限制
- 设置WinDbg的符号服务器和本地缓存目录`SRV*D:\LocalSymbols*http://msdl.microsoft.com/download/symbols`
- Attach to kernel-COM-能选的对勾都选上-填波特率-Port填`\\.\pipe\Windows7x86`
- 点击OK以调试虚拟机内核
- 设置WinDbg的符号服务器代理`set _NT_SYMBOL_PROXY=代理服务器地址:端口号`
