---
title: Windows下用vscode编写python的问题总结
description: 记录一下在Windows写Python代码遇到的问题，当然是很简单的一些小问题，算是本地备份一下笔记
date: 2021-01-19 17:48:00+0800
categories:
    - 软件指南
tags:
    - Windows
    - Python
    - vscode
---

本文来源：[Moeomu的博客](/zh-cn/posts/windows下用vscode编写python的问题总结/)

## VSCode-Python-Venv-PowerShell未签名环境无法激活

> 在Windows下激活VENV虚拟Python环境写代码遇到了PowerShell文件的无法运行的事情，原因是PowerShell脚本文件未签名，百度寻找了一下解决办法却是清一色的让将Windows安全策略改成签名，如下

- `set-executionpolicy remotesigned`

> 总觉得不应该这样，这样修改策略实属下策，去谷歌找了一下解决办法，发现这是个2018年的问题，未有良好的解决办法，但是仍旧找到了一份好的策略，即将当前用户的签名策略改成需远程签名，其它用户仍旧是阻挡

- `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`

> 随后可以通过以下代码查看策略修改情况

- `Get-ExecutionPolicy -LIST`

## Python更换PIP源

> 每次都搜索引擎找源太麻烦了，不如直接记个笔记备份到本地为好

### Windows

`%HOMEPATH%/pip/pip.ini`

### Linux & macOS

`~/.pip/pip.conf`

### 编辑格式

```text
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
```

### pip源

| 名称 | 源的地址 |
| - | - |
| 清华 | [https://pypi.tuna.tsinghua.edu.cn/simple](https://pypi.tuna.tsinghua.edu.cn/simple) |
| 阿里云 | [http://mirrors.aliyun.com/pypi/simple/](http://mirrors.aliyun.com/pypi/simple/) |
| 中国科技大学 | [https://pypi.mirrors.ustc.edu.cn/simple/](https://pypi.mirrors.ustc.edu.cn/simple/) |
| 华中理工大学 | [http://pypi.hustunique.com/](http://pypi.hustunique.com/) |
| 山东理工大学 | [http://pypi.sdutlinux.org/](http://pypi.sdutlinux.org/) |
| 豆瓣 | [http://pypi.douban.com/simple/](http://pypi.douban.com/simple/) |

## pip模块未找到错误

- `python -m ensurepip`
- `python -m pip install --upgrade pip`
