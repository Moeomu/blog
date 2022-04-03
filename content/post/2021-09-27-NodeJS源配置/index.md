---
title: NodeJS source configuration
description: NodeJS source configuration
date: 2021-09-27 13:56:01+0800
categories:
    - Config
tags:
    - Programming
    - NodeJS
---

Source: [Moeomu's blog](/posts/nodejs-source-configuration/)

## Installation

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

## Configure mirror source

### Set as Taobao mirror source

```shell
npm config set registry https://registry.npm.taobao.org/
yarn config set registry https://registry.npm.taobao.org/
```

### Reset to official mirror source

```shell
npm config set registry https://registry.npmjs.org/
yarn config set registry https://registry.yarnpkg.com/
```
