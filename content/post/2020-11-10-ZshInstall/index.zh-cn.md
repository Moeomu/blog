---
title: Zsh Install
description: 关于zsh的安装和配置
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

本文来源：[Moeomu的博客](/zh-cn/posts/zsh-install/)

## 安装zsh(install zsh)

- macOS: `brew install zsh`
- Linux
  - Arch Linux: `sudo pacman -S zsh`
  - Ubuntu: `sudo apt install zsh`

## 克隆Git仓库(clone git repo)

- China mainland: `git clone https://gitee.com/mirrors/oh-my-zsh.git`
- Global: `git clone https://github.com/ohmyzsh/ohmyzsh.git`

## 配置zsh(config)

### 设置为默认shell(set default shell)

- Set default shell: `chsh -s /usr/bin/zsh`
- Rename: `mv oh-my-zsh .oh-my-zsh`
- Set user profile: `cp .oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc`

### 自动补全插件(auto suggestions plugin)

- `git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions`
- edit `~/.zshrc`:
  - `plugins=(git)`=>`plugins=(git zsh-autosuggestions)`