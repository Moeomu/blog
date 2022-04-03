---
title: 虚拟机硬件加速导致的Electron应用白屏的问题
description: 虚拟机硬件加速导致的Electron应用白屏的问题
slug: vm-hardware-acceleration-causes-electron-app-white-screen
date: 2021-10-12 19:22:01+0800
categories:
    - VM
    - Solution
tags:
    - macOS
    - parallels
---

本文来源：[Moeomu的博客](/p/vm-hardware-acceleration-causes-electron-app-white-screen/)

## 问题再现

- 可重现：是
- 主机系统：macOS BigSur 11.6
- 虚拟机系统：Ubuntu Desktop 20.04.3
- Parallels虚拟机版本：17.0.1
- 问题描述：虚拟机系统在安装或者运行基于NodeJS尤其是Electron的应用后，应用会出现占据1/4的白色屏幕且3/4黑屏的异常显示情况，无法正常使用
- 故障典型应用：
  - VSCode
  - Motrix
  - Typora

![vscode](https://i.loli.net/2021/10/13/WvksDr9PTQFi3ut.png)

![typora](https://i.loli.net/2021/10/13/bRWZPJSQqjhFUNG.png)

## 解决方案

### 方案一

给每一个Electron应用添加启动时参数`--disable-gpu`

> 补充：可以在桌面建立一个快捷方式实现，以vscode为例，内容如下
> 
> code.desktop

```shell
[Desktop Entry]
Name=Visual Studio Code
Comment=Code Editing. Redefined.
GenericName=Text Editor
Exec=/usr/share/code/code --disable-gpu --unity-launch %F
Icon=com.visualstudio.code
Type=Application
StartupNotify=false
StartupWMClass=Code
Categories=Utility;TextEditor;Development;IDE;
MimeType=text/plain;inode/directory;application/x-code-workspace;
Actions=new-empty-window;
Keywords=vscode;

X-Desktop-File-Install-Version=0.24

[Desktop Action new-empty-window]
Name=New Empty Window
Exec=/usr/share/code/code --disable-gpu --new-window %F
Icon=com.visualstudio.code
```

### 方案二

关闭Parallels/其他虚拟化软件-此虚拟机的图形-硬件加速功能，如下图所示

![close](https://i.loli.net/2021/10/13/vSLmaJbtXiBd3xR.png)

## 其他信息

Electron的官方开发人员在[Github](https://github.com/electron/electron/issues/5257#issuecomment-213890151)上提到

> You can probably disable GPU acceleration to work around this, or just use another visual machine software. Basically the GPU acceleration of Linux in virtual machines is a mess, depending on the software of visual machine, the version and distribution of Linux, and the version of Chromium, you can get various results and bugs.

- Linux的虚拟机硬件加速一团糟，因此出现这个情况是正常的
