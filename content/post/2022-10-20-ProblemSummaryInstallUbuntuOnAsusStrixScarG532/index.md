---
title: Problem summary of installing Ubuntu on AsusStrixScarG532
description: Problem summary of installing Ubuntu on AsusStrixScarG532
date: 2022-10-20 13:22:01+0800
categories:
    - Linux
    - Solution
tags:
    - Ubuntu
    - Linux
---

Source of this article: [MoeomuBlog](/posts/problem-summary-of-installing-ubuntu-on-asusstrixscarg532/)

## System sound problems

### No sound

1. `/etc/modprobe.d/alsa-base.conf`
2. Append a line at the end of this file:`options snd-hda-intel model=asus-zenbook`
3. reboot

### Sound is not adjustable

> Description: The sound is not adjustable, can only be 0 or maximum.

1. Edit `/usr/share/pulseaudio/alsa-mixer/paths/analog-output.conf.common`
2. Add the following to the front of `[Element PCM]`

  ```text
  [Element Master]
  switch = mute
  volume = ignore
  ```

3. Kill pulse audio daemon and it will automatically restart: `sudo pulseaudio -k`

## Suspend problems

### Can't wake up again after locking the screen

> Description: After locking the screen, the computer can't wake up again. The hard drive and other hardware work fine, but the screen is all black.

1. install laptop-mod-tools: `sudo apt install laptop-mode-tools`
2. start laptop-mode-tools: `sudo laptop_mode start`
3. add mutter debug environment variable, edit `/etc/environment`
   - if in Ubuntu 22.04: `MUTTER_DEBUG_ENABLE_ATOMIC_KMS=0`
   - if in Ubuntu 22.10 and later: `MUTTER_DEBUG_FORCE_KMS_MODE=simple`
4. reboot

## reference

- [snd-hda-intel model for UBUNTU 20.04 installed in an ASUS ROG STRIX G15 (G512) - AskUbuntu](https://askubuntu.com/questions/1288054/snd-hda-intel-model-for-ubuntu-20-04-installed-in-an-asus-rog-strix-g15-g512)
- [Volume either muted or maxed out, nothing in between - ubuntu forums](https://ubuntuforums.org/showthread.php?t=2414755)
- [Blanked screen doesn't wake up after locking - Launchpad](https://bugs.launchpad.net/ubuntu/+source/mutter/+bug/1968040)
