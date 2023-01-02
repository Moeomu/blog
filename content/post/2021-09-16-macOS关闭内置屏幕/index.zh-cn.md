---
title: macOS关闭内置屏幕
description: macOS关闭内置屏幕
date: 2021-09-16 17:00:00+0800
categories:
    - 软件指南
tags:
    - macOS
---

本文来源：[Moeomu的博客](/zh-cn/posts/macos关闭内置屏幕/)

## 磁铁大法

寻找两块磁铁，放置在 MacBook的音响附近的位置，欺骗系统合盖检测从而实现关闭内置屏幕

## NVRAM启动设置法

> 此方法从驱动层面实现了将画面单独显示在外接屏幕上

### 启用

- 重新启动，按着`Command+R`按键不放
- 输入密码进入启动设置-终端
- 输入命令`nvram boot-args="niog=1"`
- 连接外接显示器，连接电源，重新启动
- 启动后输入用户密码立刻合盖，随后看到外接屏有画面，打开MacBook的盖子，成功

### 还原

- 重新启动，按着`Command+R`按键不放
- 输入密码进入启动设置-终端
- 输入命令`nvram -d boot-args`
- 重新启动
