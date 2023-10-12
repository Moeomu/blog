---
title: 在一台VPS上部署Docker服务
description: 所有服务部署在Docker中，服务与服务之间通讯使用Docker Network，Caddy反向代理所有服务，访客直接通过Cloudflare CDN访问。
date: 2022-10-24 22:03:01+0800
categories:
    - 软件指南
tags:
    - Docker
    - Caddy
    - VPS
image: https://cdn.statically.io/gh/Misakaou/imagestorage@master/20221026/00002-2908894140.26d7gclffj8g.webp
---

本文来源: [MoeomuBlog](/zh-cn/posts/在一台vps上部署docker服务/)

## 背景故事

- 有一些开源的服务非常想体验一下，但是无法部署在Paas这种服务上，或者能部署但是无法持久化存储一些文件。所以不久前自己购买了一个VPS，考虑到安全性使用Docker来部署所有服务，而本地的反向代理使用Caddy，而Caddy通过Cloudflare API自动从受Cloudflare CDN保护的DNS解析下获取和续期HTTPS证书。以下是我部署服务的全部过程，如果有错误和疏漏请一定指出。
- 本文将不会详细到每一条命令，而是描述一个大体的构建框架思路。

> 警告：如果忽略本文中具体段落的阅读官方文档的建议直接执行接下来的指令，很大概率会出错。这时候可以去仔细查看那一段提到的官方文档，或者在本文的评论区提问。

## 创建Docker基本服务框架

### 创建用户

1. 运行以下命令

   ```sh
    useradd -m mydocker
    usermod -s /bin/bash mydocker
    su mydocker
    curl -fsSL https://get.docker.com/rootless | sh
   ```

2. 遵循Docker官方的[Run the Docker daemon as a non-root user (Rootless mode) - Docker Document](https://docs.docker.com/engine/security/rootless/)指南将您希望的配置项配置好

### 创建Docker网络

- `docker network create caddy`

## 部署Web服务

### 部署Alist服务

> Alist的Github开源地址：[alist - Github](https://github.com/alist-org/alist)

- Alist部署指令
  - `-d`表示在后台运行
  - `--name`指定此Docker服务名称
  - `-p 宿主机端口:容器端口`指定绑定的端口
  - `--network=caddy`表示绑定的Docker Network
  - `-v 宿主机路径:容器内路径`指定需要永久存储的文件位置
  - `--restart=always`表示服务停机重启。

```sh
docker run -d --name="alist" \
    -p 5244:5244 \
    --network=caddy \
    -v $HOME/docker_data/alist:/opt/alist/data \
    --restart=always \
    xhofe/alist:latest
```

- 更多Alist参考指令
  - 查看Alist密码：`docker exec -it alist ./alist password`
  - 查看Alist日志：`docker logs -f alist`

### 部署Whoogle服务

- 部署指令如下，指令详情不再解释，和上文雷同

```sh
docker run -d  --name whoogle \
    -p 5000:5000 \
    --network=caddy \
    --restart=always \
    benbusby/whoogle-search:latest
```

## 部署Caddy

### 先决条件

在使用Docker部署Caddy服务之前，需要先创建所需要的Caddyfile配置文件，以供Caddy服务自动配置使用。

### Caddyfile

1. 创建永久存储位置：`mkdir -p $HOME/docker_data/caddy/`
2. 创建并编辑Caddyfile文件：`vim $HOME/docker_data/caddy/Caddyfile`

> Caddyfile

```Caddyfile
(site_config) {
    tls me_tls@example.com {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    # more config

    reverse_proxy {args.0}
}

drive.example.com {
    import site_config alist:5244
}

search.example.com {
    import site_config whoogle:5000
}
```

> 补充说明：`alist:5244`和`whoogle:5000`能正常工作并正常被解析的原因是：这三个容器都存在于同一个Docker Network中，因此可以通过容器名称进行解析。

### 使用Docker部署Caddy

- 部署指令如下，指令详情不再解释，和上文雷同
- 以下需要特别注意的是
  - `your_cloudflare_login_email@example.com`替换为你的cloudflare账号邮箱
  - `your_cloudflare_api_token`替换为你的cloudflare api token，你可以遵循[Create an API token - Cloudflare Docs](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)指南获取你的api token，注意这里创建的时候一定要选择`Zone / DNS / Edit`和`Zone / Zone / Read`权限

```sh
docker run -d --name="caddy" \
    -p 443:443 \
    --network=caddy \
    -v $HOME/docker_data/caddy/Caddyfile:/etc/caddy/Caddyfile \
    -v $HOME/docker_data/caddy/data:/data \
    -v $HOME/docker_data/caddy/config:/config \
    -e CLOUDFLARE_EMAIL=your_cloudflare_login_email@example.com \
    -e CLOUDFLARE_API_TOKEN=your_cloudflare_api_token \
    -e ACME_AGREE=true \
    --restart=always \
    slothcroissant/caddy-cloudflaredns:latest
```

## 参考文献

- Run the Docker daemon as a non-root user (Rootless mode). (2022, September 22). Docker Documentation. <https://docs.docker.com/engine/security/rootless/>
- Docker Hub. (n.d.). Hub.docker.com. Retrieved October 24, 2022, from <https://hub.docker.com/r/slothcroissant/caddy-cloudflaredns>
- Unrecognized directive: dns. (2020, May 14). Caddy Community. <https://caddy.community/t/unrecognized-directive-dns/8149/9>
- Server, C. W. (n.d.). Caddyfile Concepts - Caddy Documentation. Caddyserver.com. Retrieved October 24, 2022, from <https://caddyserver.com/docs/caddyfile/concepts>
