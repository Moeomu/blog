---
title: Homebrew and Brewcask installation
description: HomeBrew installation, really experienced a number of failures, here to summarize the success of the talk
date: 2020-11-10 19:43:00+0800
categories:
    - InstallationGuide
tags:
    - macOS
    - Homebrew
---

Please note the timeliness:**This article was written on 2020/11/10**&&**This article was updated on 2021/07/02**

Source of this article: [Moeomu's blog](/posts/homebrew-and-brewcask-installation/)

## Install git

- Method 1: Install XCode Terminal:`xcode-select --install`
- Method 2: Go directly to the git website and download the macOS installer for git

## Install HomeBrew

- First download the original [installer.sh](https://cdn.jsdelivr.net/gh/Homebrew/install@master/install.sh) jsDelivr CDN image
- Run it by entering the following command.
  
  ```shell
  git config --global url."https://mirrors.ustc.edu.cn/homebrew-core.git".insteadOf "https://github.com/Homebrew/homebrew-core"
  git config --global url."https://mirrors.ustc.edu.cn/linuxbrew-core.git".insteadOf "https://github.com/Homebrew/linuxbrew-core"
  git config --global url."https://mirrors.ustc.edu.cn/brew.git".insteadOf "https://github.com/Homebrew/brew"
  chmod +x install.sh
  HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles ./install.sh
  ```

- Cloning the Homebrew-Cask project via USTC's git repository
  - `cd /usr/local/Homebrew/Library/Taps/homebrew`
  - `git clone https://mirrors.ustc.edu.cn/homebrew-cask.git`

## Check source configuration

- Checking the brew mirror source: `git -C "$(brew --repo)" remote -v`
- View homebrew-core mirror source: `git -C "$(brew --repo homebrew/core)" remote -v`
- View homebrew-cask mirror sources: `git -C "$(brew --repo homebrew/cask)" remote -v` 
- The qualifying criteria is that the addresses of these sources are all USTC URLs

## Update

- `brew update`

## Install Oh-My-Zsh (not required)

- Update zsh: `brew install zsh`
- git clone oh-my-zsh project: `git clone https://gitee.com/mirrors/oh-my-zsh.git`
- Rename the project: `mv oh-my-zsh .oh-my-zsh`
- Copy the template to the home directory named .zshrc: `cp .oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc`
- Exit Terminal, start Terminal, success

## Configure Terminal (not required)

- Preferences-On startup, open-New window with description file-Homebrew
- Preferences-Description File-Homebrew-Text-Font-14 points

## Configure Terminal agent (not required)

```shell
export https_proxy=http://127.0.0.1:7809
export http_proxy=http://127.0.0.1:7809
```
