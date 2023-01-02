---
title: OhMyZsh安装指南
description: 关于zsh的安装和配置
date: 2020-11-10 19:43:00+0800
categories:
    - 软件指南
tags:
    - macOS
    - Linux
    - zsh
---
 
zsh install and config

本文来源：[Moeomu的博客](/zh-cn/posts/ohmyzsh安装指南/)

## 安装zsh

- macOS: `brew install zsh`
- Linux
  - Arch Linux: `sudo pacman -S zsh`
  - Ubuntu: `sudo apt install zsh`

## 克隆Git仓库(clone git repo)

- 中国大陆: `git clone https://gitee.com/mirrors/oh-my-zsh.git`
- 世界: `git clone https://github.com/ohmyzsh/ohmyzsh.git`

## 配置zsh(config)

### 设置为默认shell(set default shell)

- Set default shell: `chsh -s /usr/bin/zsh`
- Rename: `mv oh-my-zsh .oh-my-zsh`
- Set user profile: `cp .oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc`

### 自动补全插件(auto suggestions plugin)

- `git clone https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions`
- edit `~/.zshrc`:
  - `plugins=(git)`=>`plugins=(git zsh-autosuggestions)`
