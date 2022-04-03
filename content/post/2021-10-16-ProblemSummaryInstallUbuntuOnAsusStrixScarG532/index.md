---
title: Problem summary of installing Ubuntu on AsusStrixScarG532
description: Problem summary of installing Ubuntu on AsusStrixScarG532
slug: installing-ubuntu-on-AsusStrixScarG532
date: 2021-10-16 11:22:01+0800
categories:
    - Linux
    - Solution
tags:
    - Ubuntu
    - Linux
    - Driver
---

Source of this article: [MoeomuBlog](/p/installing-ubuntu-on-AsusStrixScarG532/)

## Install Nvidia driver on Ubuntu with SecurityBoot

- Install Nvidia graphic card driver during install ubuntu
  1. During installation, make sure to select the `Install third-party software for graphics and Wi-Fi hardware and addition media formats` in `Updates and other software` screen.
  2. Select `Configure Secure Boot`, and set password.
  3. Continue Ubuntu installation as normal.
  4. During the first reboot, `Perform MOK management` screen will showup. Select `Enroll MOK` option.
  5.  Select `Continue`, then, `Yes`.
  6. `Enroll the key(s)?` screen will present. Enter the password from Step 2.
  7. OK and reboot.
- Install the "third party" software option after skipped it in the ubuntu installer
  1. `sudo apt-get install ubuntu-restricted-extras`
  2. Refer to above

## Sound Not Working

1. edit`/etc/modprobe.d/alsa-base.conf`
2. Add a line `options snd-hda-intel model=asus-zenbook` at the end of the file