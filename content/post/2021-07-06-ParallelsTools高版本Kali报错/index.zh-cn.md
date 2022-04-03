---
title: ParallelsTools高版本Kali报错
description: macOS下的虚拟机Parallels的工具ParallelsTools安装于高版本Kali报错的解决办法
date: 2021-07-06 15:55:00+0800
categories:
    - Solution
tags:
    - macOS
    - parallels
---

本文来源：[Moeomu的博客](/zh-cn/posts/ParallelsTools高版本Kali报错/)

## 首先更新kali源

- 阿里云源如下

```sourcelist
deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
```

## 报错内容

```shell
An error occurred while installing the following packages:
- libelf-dev
- linux-headers-5.10.0-kali9-amd64
- dkms

Install these packages manually and start the Parallels Tools installation again.
```

## 解决方案

- 安装缺失的包

```shell
sudo apt install dkms
sudo apt intall libelf-dev
```

> 坑点记录：  
> Kali 2021默认的`xfce`桌面环境会导致安装ParallelsTools后白屏，所以要事先将桌面环境切换为`gnome`或者在安装kali的时候就取消选择`xface`，选中`gnome`
