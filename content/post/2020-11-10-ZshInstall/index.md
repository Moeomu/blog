---
title: Zsh Installation
description: about zsh installation and configuration
date: 2020-11-10 19:43:00+0800
categories:
    - InstallationGuide
    - Config
tags:
    - macOS
    - Linux
    - zsh
---

zsh install and config

Source: [Moeomu's blog](/posts/zsh-installation/)

## Install zsh

- macOS: `brew install zsh`
- Linux
  - Arch Linux: `sudo pacman -S zsh`
  - Ubuntu: `sudo apt install zsh`

## Clone git repo

- China mainland: `git clone https://gitee.com/mirrors/oh-my-zsh.git`
- Global: `git clone https://github.com/ohmyzsh/ohmyzsh.git`

## config zsh

### set as default shell

- Set default shell: `chsh -s /usr/bin/zsh`
- Rename: `mv oh-my-zsh .oh-my-zsh`
- Set user profile: `cp .oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc`

### auto suggestions plugin

- `git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions`
- edit `~/.zshrc`:
  - `plugins=(git)`=>`plugins=(git zsh-autosuggestions)`
