---
date: "2026-02-03T14:28:44+08:00"
title: "Docker -- docker compose"
tags: []
categories: "笔记"
description: ""
draft: true
searchHidden: false

showToc: true
TocOpen: true
hidemeta: false
comments: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: false
UseHugoToc: true
---

`docker compose` 是 `Docker` 官方提供的容器编排工具，用以简化多容器应用的生命周期管理。

<!--more-->

通过一个 `yaml` 配置即可定义一个多容器应用，配合 `docker compose` 命令，既可方便的完成应用的创建、启动、停止、销毁等操作。

## 核心概念

### 服务 Services

对应一个或者一组相同的容器，是 `compose` 配置的核心，应用中每一个独立组件（如 Web服务、 MySQL数据库、 Redis缓存）都可以定义为一个服务。  
每个服务可配置镜像、容器端口、环境变量、依赖关系、启动命令等细节，`compose` 会根据配置自动创建并管理对应容器。

### 网络 Networks

用于容器间的网络通信，`compose` 会创建一个默认的桥接网络，应用所用到的所有容器都会加入到该网络。  
容器间可以使用服务名互相访问，当然也可手动定义自定义网络，实现网络隔离、跨项目通信等需求。

### 数据卷 Volumes

用于容器数据的持久化存储，解决容器重启或销毁后数据丢失的问题。数据卷即可以通过 `yaml` 配置文件定义，也可直接绑定挂载到宿主机上。

## 快速使用

首先，我们需要通过一个配置文件定义我们的应用。官方推荐使用 `compose.yaml` 作为文件名，如果执行 `docker compose` 命令时未指定配置文件，`compose` 会在当前目录优先寻找 `compose.yaml` 文件作为默认配置（此外 `compose.yml`、 `docker-compose.yaml`、 `docker-compose.yml` 也会被尝试，但优先级低于 `compose.yaml`）。

这里以创建一个 `Nginx` 应用作为示例，`compose.yaml` 文件内容如下：

```yaml
# compose.yaml
name: "web"
services:
  nginx:
    image: "nginx:1.28.1-alpine-slim"
```

- `name` 顶级元素，用来定义应用（项目）的名称。
- `services` 顶级元素，用来定义服务。可同时包含多个服务。
  - `nginx` 自定义的服务名称，可根据需求修改。
  - `image` 使用到的镜像。

我们可以通过 `docker compose up` 前台启动应用：

```bash
docker compose up
[+] Running 8/8
 ✔ nginx Pulled                                                                                                                                                           17.3s
   ✔ 589002ba0eae Pull complete                                                                                                                                            4.2s
   ✔ 7f04ac50f331 Pull complete                                                                                                                                            4.2s
   ✔ 165d5d75c0d4 Pull complete                                                                                                                                            4.3s
   ✔ 55e0aa84454e Pull complete                                                                                                                                            4.3s
   ✔ 989471907e18 Pull complete                                                                                                                                            5.5s
   ✔ df98287c1c7d Pull complete                                                                                                                                            7.4s
   ✔ 688c08f786c6 Pull complete                                                                                                                                            7.5s
[+] Running 2/2
 ✔ Network web_default    Created                                                                                                                                          0.0s
 ✔ Container web-nginx-1  Created                                                                                                                                          0.0s
Attaching to nginx-1
nginx-1  | /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
nginx-1  | /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
nginx-1  | /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
...
```

可以看到 `docker compose` 首先为我们拉取了镜像，然后创建了一个默认桥接网络 `web_default`, 最后启动了 `nginx` 容器。

使用 `docker compose up` 启动的应用一旦我们关闭终端或使用 `ctrl + c` 结束进程，应用就会停止（停止所有容器），我们可以使用 `-d` 参数让应用运行于后台。

```bash
docker compose up -d
[+] Running 1/1
 ✔ Container web-nginx-1  Started
```

使用 `docker compose ps` 命令我们可以查看当前应用运行中的容器。

```bash
docker compose ps
NAME          IMAGE                      COMMAND                   SERVICE   CREATED         STATUS              PORTS
web-nginx-1   nginx:1.28.1-alpine-slim   "/docker-entrypoint.…"   nginx     7 minutes ago   Up About a minute   80/tcp
```

可以看到有一个容器 `web-nginx-1` 正在运行。

## 生命周期管理

`docker compose` 

## 端口映射

虽然我们已经启动的 `nginx` 容器，但我们还不能镜像访问。这是由于容器仅加入了默认桥接网络 `web_default`，需要先进行端口映射才能访问容器内的端口。

```yaml
# compose.yaml
name: "web"
services:
  nginx:
    image: "nginx:1.28.1-alpine-slim"
    ports:
      - "9999:80"
```

使用 `ports` 描述端口映射规则，格式为 `宿主机端口:容器端口`，可以同时映射多组。最后执行 `docker compose up -d` 命令，`compose` 会根据配额文件的变化进行更新。最后，我们进行一下方法测试。

```bash
curl http://localhost:9999
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

可以正常访问 `nginx` 页面，说明端口映射成功。

## 数据卷绑定

### 挂载到本地目录



## 参考资料

[Docker 官方文档](https://docs.docker.com/)
