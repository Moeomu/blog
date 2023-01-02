---
title: Homebrew和Brewcask的安装
description: HomeBrew的安装历经了多次失败，在此总结一下成功之谈
date: 2020-11-10 19:43:00+0800
categories:
    - 软件指南
tags:
    - macOS
    - Homebrew
    - Brewcask
---

大家请注意时效性：**本文写于2020/11/10**&&**本文更新于2021/07/02**

本文来源：[Moeomu的博客](/zh-cn/posts/homebrew和brewcask的安装/)

## 安装git

- 方法一：安装XCode Terminal:`xcode-select --install`
- 方法二：直接去git网站下载git的macOS安装包

## 安装HomeBrew

- 首先下载原版[installer.sh](https://cdn.jsdelivr.net/gh/Homebrew/install@master/install.sh)的jsDelivr CDN镜像
- 输入下列指令运行：
  
  ```shell
  git config --global url."https://mirrors.ustc.edu.cn/homebrew-core.git".insteadOf "https://github.com/Homebrew/homebrew-core"
  git config --global url."https://mirrors.ustc.edu.cn/linuxbrew-core.git".insteadOf "https://github.com/Homebrew/linuxbrew-core"
  git config --global url."https://mirrors.ustc.edu.cn/brew.git".insteadOf "https://github.com/Homebrew/brew"
  chmod +x install.sh
  HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles ./install.sh
  ```

- 通过USTC的git源克隆Homebrew-Cask项目
  - `cd /usr/local/Homebrew/Library/Taps/homebrew`
  - `git clone https://mirrors.ustc.edu.cn/homebrew-cask.git`

## 检查源配置

- 查看brew镜像源：`git -C "$(brew --repo)" remote -v`
- 查看homebrew-core镜像源：`git -C "$(brew --repo homebrew/core)" remote -v`
- 查看homebrew-cask镜像源：`git -C "$(brew --repo homebrew/cask)" remote -v` 
- 合格的标准是这些源的地址全是USTC的网址

## Update

- `brew update`

## 安装Oh-My-Zsh(非必须)

- 更新zsh：`brew install zsh`
- git克隆oh-my-zsh项目：`git clone https://gitee.com/mirrors/oh-my-zsh.git`
- 重命名项目：`mv oh-my-zsh .oh-my-zsh`
- 将模版复制到home目录下命名为.zshrc：`cp .oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc`
- 退出Terminal，启动Terminal，成功

## 配置Terminal(非必须)

- 偏好设置-启动时，打开-使用描述文件新建窗口-Homebrew
- 偏好设置-描述文件-Homebrew-文本-字体-14点

## 配置Terminal代理(非必须)

```shell
export https_proxy=http://127.0.0.1:7809
export http_proxy=http://127.0.0.1:7809
```
