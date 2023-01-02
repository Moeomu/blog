---
title: ParallelsDesktop16 修复网络
description: macOS-parallels desktop网络初始化失败解决办法
date: 2020-12-16 13:42:00+0800
categories:
    - 软件指南
tags:
    - macOS
    - ParallelsDesktop
---


本文来源：[Moeomu的博客](/zh-cn/posts/parallelsdesktop16-修复网络/)

## 前奏

- 在macOS Big Sur下可以使用
- 之前是使用命令启动的不完美解决办法，还会导致一系列权限问题，现在终于完美了

## 解决方案

- 最高权限编辑文件`/Library/Preferences/Parallels/network.desktop.xml`
- 将`UseKextless`字段的值从`-1`改为`0`

> 注意：这个值可能并不是所有人-1，也可能并不是所有人改为0都行，勇于尝试嘛
