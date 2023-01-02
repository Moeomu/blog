---
title: Windows下使用GPG为Git Commit签名
description: Windodws下的GitGPG密钥配置和开启签名验证
date: 2021-06-05 08:50:00+0800
categories:
    - 软件指南
tags:
    - Windows
    - Git
    - GPG
---

本文来源：[Moeomu的博客](/zh-cn/posts/windows下使用gpg为git-commit签名/)

## 下载GPG4WIN

下载链接：[gpg4win](https://www.gpg4win.org/thanks-for-download.html)

## 创建和应用GPG密钥

### 创建GPG密钥

- 创建：`gpg --full-generate-key`
- 密钥长度：`4096`
- 输入用户名、邮箱
- 列出所有密钥：`gpg --list-secret-keys --keyid-format=long`
- 根据keyid导出密钥：`gpg --armor --export KEYID`

### 应用密钥

- 将密钥导入Github和Gitee

## 配置Git Windows

- 配置默认用户名和邮箱，需要和创建GPG时设定的值保持一致
  - `git config --global user.name USERNAME`
  - `git config --global user.email EMAIL`
- 配置密钥
  - `git config --global user.signingKey KEYID`
- 启用全局加密签名
  - `git config --global commit.gpgSign true`
