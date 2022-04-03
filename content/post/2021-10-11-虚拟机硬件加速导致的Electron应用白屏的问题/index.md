---
title: Virtual machine hardware acceleration causes Electron application white screen problem
description: Virtual machine hardware acceleration causes Electron application white screen problem
date: 2021-10-12 19:22:01+0800
categories:
    - VM
    - Solution
tags:
    - macOS
    - parallels
---

Source: [Moeomu's blog](/posts/virtual-machine-hardware-acceleration-causes-electron-application-white-screen-problem/)

## Problem Reproducible

- Reproducible: Yes
- Host system: macOS BigSur 11.6
- Virtual machine system: Ubuntu Desktop 20.04.3
- Parallels Virtual Machine version: 17.0.1
- Description of the problem: After installing or running NodeJS-based applications, especially Electron, the application will take up 1/4 of the white screen and 3/4 of the black screen with abnormal display, which cannot be used normally
- Typical application failure.
  - VSCode
  - Motrix
  - Typora

![vscode](https://i.loli.net/2021/10/13/WvksDr9PTQFi3ut.png)

![typora](https://i.loli.net/2021/10/13/bRWZPJSQqjhFUNG.png)

## Solutions

### Solution 1

Add a startup parameter `-disable-gpu` to each Electron application

> Add: This can be achieved by creating a shortcut on the desktop, using vscode as an example, with the following content
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

### Solution 2

Turn off the graphics-hardware acceleration for Parallels/other virtualization software - this virtual machine, as shown below

![close](https://i.loli.net/2021/10/13/vSLmaJbtXiBd3xR.png)

## Additional Information

The official developers of Electron mention on [Github](https://github.com/electron/electron/issues/5257#issuecomment-213890151)

> You can probably disable GPU acceleration to work around this, or just use another visual machine software. Basically the GPU acceleration of Linux in virtual machines is a mess, depending on the software of visual machine, the version and distribution of Linux, and the version of Chromium, you can get various results and bugs.

- Linux's virtual machine hardware acceleration is a mess, so it's normal for this to happen
