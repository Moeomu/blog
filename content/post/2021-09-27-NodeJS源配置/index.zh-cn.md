---
title: NodeJS源配置
description: NodeJS源配置
date: 2021-09-27 13:56:01+0800
categories:
    - Config
tags:
    - Programming
    - NodeJS
---

本文来源：[Moeomu的博客](/zh-cn/posts/nodejs源配置/)

## 安装

> Debain(Ubuntu, etc.)

```shell
apt install node
apt install yarn
```

> macOS

```shell
brew install node
brew install yarn
```

## 配置镜像源

### 设置为淘宝镜像源

```shell
npm config set registry https://registry.npm.taobao.org/
yarn config set registry https://registry.npm.taobao.org/
```

### 重置为官方镜像源

```shell
npm config set registry https://registry.npmjs.org/
yarn config set registry https://registry.yarnpkg.com/
```
