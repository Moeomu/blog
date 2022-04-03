---
title: ProblemsEncounteredInInstallingArchonROGLaptop
description: Problems encountered in installing Arch on ROG Laptop
slug: installing-arch-on-ROGLaptop
date: 2021-11-09 09:22:01+0800
categories:
    - Linux
    - Solution
tags:
    - Arch
    - Linux
    - Driver
---

Source of this article: [MoeomuBlog](/p/installing-arch-on-ROGLaptop/)

## No display after installing NVIDIA graphics card

### Installed packages

- `xf86-video-intel`
- `mesa`
- `nvidia`

### Solution

- remove package: `sudo pacman -R xf86-video-intel`
- remove xconfig: `sudo rm /etc/X11/xorg.conf`
- edit xconfig: `sudo vim /etc/X11/xorg.conf.d/10-nvidia-drm-outputclass.conf`

> 10-nvidia-drm-outputclass.conf

```text
Section "OutputClass"
    Identifier "intel"
    MatchDriver "i915"
    Driver "modesetting"
EndSection

Section "OutputClass"
    Identifier "nvidia"
    MatchDriver "nvidia-drm"
    Driver "nvidia"
    Option "AllowEmptyInitialConfiguration"
    Option "PrimaryGPU" "yes"
    ModulePath "/usr/lib/nvidia/xorg"
    ModulePath "/usr/lib/xorg/modules"
EndSection
```

- reboot

> More references: [ArchWiki-PRIME](https://wiki.archlinux.org/title/PRIME)

## No sound

- edit file:`sudo vim /etc/modprobe.d/alsa-base.conf`

> alsa-base.conf

```text
options snd-hda-intel model=asus-zenbook
```

- Problems that still exist: The volume can only be 0% or 100%, but the application volume can be adjusted individually.

> More references: [No sound ALC294 Asus ROG Strix 512 Ubuntu 20.04.01](https://askubuntu.com/questions/1276428/no-sound-alc294-asus-rog-strix-512-ubuntu-20-04-01)

## The wireless network is unstable

### Installed packages

- `dhcpcd`
- `NetworkManager`
- `iwd`

### Solution

- stop and disable service:
  - `sudo systemctl stop dhcpcd`
  - `sudo systemctl disable dhcpcd`
  - `sudo systemctl stop iwd`
  - `sudo systemctl disable iwd`
- remove package: `sudo pacman -R dhcpcd`
- reboot

### Fundamental problem

The fundamental problem is that dhcpcd conflicts with NetworkManager.
