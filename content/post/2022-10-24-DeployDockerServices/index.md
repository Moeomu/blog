---
title: Deploying Docker services on a VPS
description: All services are deployed in Docker, services communicate with each other using Docker Network, Caddy reverse proxy all services, and visitors access directly through Cloudflare CDN.
date: 2022-10-24 22:03:01+0800
categories:
    - SoftwareGuide
tags:
    - Docker
    - Caddy
    - VPS
image: https://cdn.staticaly.com/gh/Misakaou/imagestorage@master/20221026/00002-2908894140.26d7gclffj8g.webp
---

Source of this article: [MoeomuBlog](/posts/deploying-docker-services-on-a-vps/)

## Pre Introduction

- There are some open source services that I really want to experience, but can not be deployed on a service like Paas, or can be deployed but can not persistently store some files. So not long ago, I bought a VPS, considering the security of the use of Docker to deploy all services, and the local reverse proxy using Caddy, and Caddy through the Cloudflare API automatically from the Cloudflare CDN protected DNS resolution under the acquisition and renewal of HTTPS certificates. The following is the entire process of deploying my service, if there are errors and omissions please be sure to point out.
- This article will not go into detail to each command, but describe a general idea of the framework for building.

> Warning: If you ignore the advice to read the official documentation in specific paragraphs of this article and execute the next commands directly, there is a high probability that you will make a mistake. At this point, you can go through the official documentation mentioned in that paragraph, or ask questions in the comments section of this article.

## Create Docker basic service framework

### Create a user

1. Run the following command

   ```sh
    useradd -m mydocker
    usermod -s /bin/bash mydocker
    su mydocker
    curl -fsSL https://get.docker.com/rootless | sh
   ```

2. Follow Docker's official [Run the Docker daemon as a non-root user (Rootless mode) - Docker Document](https://docs.docker.com/engine/security/rootless/) guide Configure the configuration items you want

### Create a Docker network

- `docker network create caddy`

### Deploy the web service

### Deploy the Alist service

> Alist's Github open source address: [alist - Github](https://github.com/alist-org/alist)

- Alist deployment commands
  - `-d` means run in the background
  - `--name` specifies the name of this Docker service
  - `-p host port:container port` specifies the port to bind to
  - `--network=caddy` indicates the bound Docker Network
  - `-v host path:container path` specifies the location of the file to be stored permanently
  - `--restart=always` indicates that the service is down for restart.

```sh
docker run -d --name="alist" \
    -p 5244:5244 \
    --network=caddy \
    -v $HOME/docker_data/alist:/opt/alist/data \
    --restart=always \
    xhofe/alist:latest
```

- More Alist reference commands
  - View Alist password: `docker exec -it alist . /alist password`
  - View Alist logs: ``docker logs -f alist``

### Deploy Whoogle service

- Deploy the following commands, the details of which are not explained, and are the same as above

```sh
docker run -d --name whoogle \
    -p 5000:5000 \
    --network=caddy \
    --restart=always \
    benbusby/whoogle-search:latest
```

## Deploying Caddy

### Prerequisites

Before deploying the Caddy service using Docker, you need to create the required Caddyfile configuration file for the Caddy service to use for automatic configuration.

### Caddyfile

1. Create the permanent storage location: `mkdir -p $HOME/docker_data/caddy/`
2. Create and edit the Caddyfile file: `vim $HOME/docker_data/caddy/Caddyfile`

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

> Additional note: The reason why `alist:5244` and `whoogle:5000` work and are resolved properly is that all three containers exist in the same Docker Network, so they can be resolved by container name.

### Deploying Caddy with Docker

- The deployment command is as follows, the details of the command are not explained, it is the same as above
- The following special note is needed
  - `your_cloudflare_login_email@example.com` replaced with your cloudflare account email
  - `your_cloudflare_api_token` is replaced with your cloudflare api token, you can follow [Create an API token - Cloudflare Docs](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) guide to get your api token, note that when creating it here you must select `Zone / DNS / Edit` and `Zone / Zone / Read` permissions

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

## Reference

- Run the Docker daemon as a non-root user (Rootless mode). (2022, September 22). Docker Documentation. <https://docs.docker.com/engine/security/rootless/>
- Docker Hub. (n.d.). Hub.docker.com. Retrieved October 24, 2022, from <https://hub.docker.com/r/slothcroissant/caddy-cloudflaredns>
- Unrecognized directive: dns. (2020, May 14). Caddy Community. <https://caddy.community/t/unrecognized-directive-dns/8149/9>
- Server, C. W. (n.d.). Caddyfile Concepts - Caddy Documentation. Caddyserver.com. Retrieved October 24, 2022, from <https://caddyserver.com/docs/caddyfile/concepts>
