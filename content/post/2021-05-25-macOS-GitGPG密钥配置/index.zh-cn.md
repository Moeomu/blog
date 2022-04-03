---
title: macOS-GitGPG密钥配置
description: macOS下的GitGPG密钥配置和开启签名验证
date: 2021-05-25 07:55:00 +0800
categories:
    - Config
tags:
    - macOS
    - Git
    - GPG
---

本文来源：[Moeomu的博客](/zh-cn/posts/macos-gitgpg密钥配置/)

## 关于GitHub GPG密钥验证

### 开启Commit签名

GitHub有个新的“警惕模式”，启用后GPG密钥需要签名认证的commit才会显示“Verity”，启用方法如下

- 首先创建GPG密钥（[GitHub官方Docs](https://docs.github.com/en/github/authenticating-to-github/managing-commit-signature-verification/generating-a-new-gpg-key)有详细方法，不再赘述）
- 列出GPG密钥的特征码：`gpg -K --keyid-format LONG`，将**keyid**记录
- 告知git使用此GPG密钥：`git config user.signingkey your_keyid`
- 本地git的用户名和邮箱需要和GPG密钥生成时填入的相同：`git config user.name name`，`git config user.email email`
- 启用本地git的Commit签名：`git config commit.gpgsign true`
- Commit签名加入-S选项：`git commit -S -m message`

### 发生致命错误-无法Commit-macOS

#### 问题如下，复现于macOS 11.3.1中

```text
error: gpg failed to sign the data
fatal: failed to write commit object
```

#### macOS中的解决办法

- 更新&安装

```shell
brew upgrade gnupg  # This has a make step which takes a while
brew link --overwrite gnupg
brew install pinentry-mac
echo "pinentry-program /usr/local/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
killall gpg-agent

git config --global gpg.program gpg  # perhaps you had this already? On linux maybe gpg2
git config --global commit.gpgsign true  # if you want to sign every commit
```

- 再次签名
- 查看commit状态：`git log --show-signature -1`
