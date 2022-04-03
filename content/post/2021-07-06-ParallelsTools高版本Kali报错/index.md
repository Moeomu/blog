---
title: ParallelsTools error on high version Kali
Description: Solution for ParallelsTools, a tool for virtual machines under macOS, to install on higher versions of Kali with an error
date: 2021-07-06 15:55:00+0800
categories:
    - Solution
tags:
    - macOS
    - parallels
---

Source: [Moeomu's blog](/posts/parallelstools-error-on-high-version-kali/)

## First update the kali source

- Ali cloud source is as follows

```sourcelist
deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
```

## Error report content

```shell
An error occurred while installing the following packages:
- libelf-dev
- linux-headers-5.10.0-kali9-amd64
- dkms

Install these packages manually and start the Parallels Tools installation again.
```

## Solution

- Install the missing package

```shell
sudo apt install dkms
sudo apt intall libelf-dev
```

> Pitfall record.  
> Kali 2021's default `xfce` desktop environment will cause a white screen after installing ParallelsTools, so switch the desktop environment to `gnome` in advance or deselect `xface` and select `gnome` when installing kali
