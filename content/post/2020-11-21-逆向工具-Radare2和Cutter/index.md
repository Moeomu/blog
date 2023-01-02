---
title: Reverse Tools Radare2 and Cutter
description: open source reverse tools-Radare2 and Cutter
date: 2020-11-21 19:04:00+0800
categories:
    - SoftwareRecommendation
tags:
    - Reverse
    - Tools
---

Source: [Moeomu's blog](/posts/reverse-tools-radare2-and-cutter/)

## Introduction

### Radare2

- r2 is a rewrite from scratch of radare in order to provide a set of libraries and tools to work with binary files.

- Radare project started as a forensics tool, a scriptable command-line hexadecimal editor able to open disk files, but later added support for analyzing binaries, disassembling code, debugging programs, attaching to remote gdb servers...

- radare2 is portable.

- To learn more you may read the official radare2 book, the source code, or browse the web for blog posts or presentations from r2con.

### Cutter

- Cutter is a free and open-source reverse engineering framework powered by radare2 . Its goal is making an advanced, customizable and FOSS reverse-engineering platform while keeping the user experience at mind. Cutter is created by reverse engineers for reverse engineers.

## Installation

> Since Cutter is a GUI-based program for Radare2, you only need to download Cutter

- This test system: macOS Catalina 10.15.7
- Github download link: [github.com/radareorg/cutter/releases](https://github.com/radareorg/cutter/releases)
- Moeomu netdisk download link (may not be the latest): [Cutter-v1.12.0-x64.dmg](https://pan.moeomu.com/Software/macOS/Tools-Reverse_Pwn/Cutter-v1.12.0-x64.macOS.dmg)

## The reason why macOS Catalina can't run Cutter

> I don't know about Windows, but it doesn't work properly on macOS Catalina after installation anyway, the solution is as follows

- Find out the reason why it doesn't work properly
- Go to Cutter's folder: `cd /Applications/Cutter.app/Contents/MacOS/`
![CD-Cutter-Folder](https://s3.ax1x.com/2020/11/21/D3Sgds.png)
- Run Cutter directly to see the cause: `. /Cutter`
![Cutter-Error-Message](https://s3.ax1x.com/2020/11/21/D3S2on.png)
- Install `gettext` to solve the problem: `brew install gettext`
![install-gettext](https://s3.ax1x.com/2020/11/21/D3SWiq.png)
