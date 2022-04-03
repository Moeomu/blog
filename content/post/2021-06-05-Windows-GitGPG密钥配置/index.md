---
title: Windows-GitGPG Key Configuration
description: GitGPG key configuration and enable signature verification under Windodws
date: 2021-06-05 08:50:00+0800
categories:
    - Config
tags:
    - Windows
    - Git
    - GPG
---

Source: [Moeomu's Blog](/posts/windows-gitgpg-key-configuration/)

## Download GPG4WIN

Download link: [gpg4win](https://www.gpg4win.org/thanks-for-download.html)

## Create and apply GPG key

### Create GPG key

- Create: `gpg --full-generate-key`
- Key length: `4096`
- Enter username, email
- List all keys: `gpg --list-secret-keys --keyid-format=long`
- Export keys according to keyid: `gpg --armor --export KEYID`

### Apply the key

- Import the key into Github and Gitee

## Configure Git Windows

- Configure the default username and email, which needs to be the same as the values set when creating GPG
  - `git config --global user.name USERNAME`
  - `git config --global user.email EMAIL`
- Configure the key
  - `git config --global user.signingKey KEYID`
- Enable global cryptographic signatures
  - `git config --global commit.gpgSign true`
