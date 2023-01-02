---
title: macOS Turn off built-in screen
description: macOS Turn off the built-in screen
date: 2021-09-16 17:00:00+0800
categories:
    - SoftwareGuide
tags:
    - macOS
---

Source: [Moeomu's blog](/posts/macos-turn-off-built-in-screen/)

## The magnet trick

Find two magnets and place them near the audio of the MacBook to trick the system into closing the lid to turn off the built-in screen.

## NVRAM boot setup method

> This method enables the screen to be displayed separately on the external screen from the driver level.

### Enable

- Reboot, press and hold `Command+R` key
- Enter the password to boot setup - terminal
- Enter the command `nvram boot-args="niog=1"`
- Connect the external monitor, connect the power supply and reboot
- Close the cover immediately after booting by entering the user password, then see the external screen with a screen, open the cover of the MacBook, success

### Restore

- Reboot, press and hold `Command+R` key
- Enter the password to boot setup - terminal
- Enter the command `nvram -d boot-args`
- Reboot
