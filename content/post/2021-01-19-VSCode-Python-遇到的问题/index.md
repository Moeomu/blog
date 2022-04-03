---
title: VSCode Python encountered problems
description: Record the problems I encountered writing Python code in Windows, of course they are simple problems, it's a local backup for my notes.
date: 2021-01-19 17:48:00+0800
categories:
    - Solution
tags:
    - Windows
    - Python
    - VSCode
---

Source: [Moeomu's blog](/posts/vscode-python-encountered-problems/)

## VSCode-Python-Venv-PowerShell unsigned environment can't be activated

> I've been looking for a solution for a while, but the solution is to change the Windows security policy to a signed one, as follows

- `set-executionpolicy remotesigned`

> I found that this is a 2018 problem and there is no good solution, but I still found a good policy that changes the current user's signature policy to require remote signing, while other users are still blocked

- `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`

> You can then view the policy changes with the following code

- `Get-ExecutionPolicy -LIST`

## Python change PIP source

> It's too much trouble to search for the source every time, so why not just take a note and back it up locally?

### Windows

`%HOMEPATH%/pip/pip.ini`

### Linux & macOS

`~/.pip/pip.conf`

### Edit format

```text
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
```

### pip source

| Site | Source |
| - | - |
| Tsinghua | [https://pypi.tuna.tsinghua.edu.cn/simple](https://pypi.tuna.tsinghua.edu.cn/simple) |
| Ali Cloud | [http://mirrors.aliyun.com/pypi/simple/](http://mirrors.aliyun.com/pypi/simple/) |
| USTC | [https://pypi.mirrors.ustc.edu.cn/simple/](https://pypi.mirrors.ustc.edu.cn/simple/) |
| HUT | [http://pypi.hustunique.com/](http://pypi.hustunique.com/) |
| SUT | [http://pypi.sdutlinux.org/](http://pypi.sdutlinux.org/) |
| DouBan | [http://pypi.douban.com/simple/](http://pypi.douban.com/simple/) |

## pip module not found error

- `python -m ensurepip`
- `python -m pip install --upgrade pip`
