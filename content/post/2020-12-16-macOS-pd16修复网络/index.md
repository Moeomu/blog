---
title: ParallelsDesktop16 repair network
description: macOS-parallels desktop network initialization failure solution
date: 2020-12-16 13:42:00+0800
categories:
    - SoftwareGuide
tags:
    - macOS
    - ParallelsDesktop
---

Source: [Moeomu's blog](/posts/parallelsdesktop16-repair-network/)

## Prelude

- Works under macOS Big Sur
- Previously it was an imperfect solution using command start, which would also lead to a series of permission problems, but now it is finally perfect

## Solution

- Edit the file `/Library/Preferences/Parallels/network.desktop.xml` with highest privileges
- Change the value of the `UseKextless` field from `-1` to `0`

> Note: This value may not be -1 for everyone, and may not be 0 for everyone, so be brave and try!
