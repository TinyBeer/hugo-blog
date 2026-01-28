---
date: "2026-01-28T11:06:28+08:00"
title: "Docker -- 快速入门"
tags: ["Docker"]
categories: "笔记"
description: ""
draft: false
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

`Docker` 是一个开源的应用容器引擎与容器化平台，它将应用运行环境同应用一起打包，使得应用的部署不再依赖具体的运行环境，从而缩短应用交付、测试、部署的时间。

<!--more-->

`Docker` 基于 `Go` 语言开发，利用 `Linux` 内核的 `Namespace`、`CGroup` 等特性实现操作系统级虚拟化，可将应用及其依赖打包成轻量、可移植的容器，实现 “一次构建，处处运行”，解决环境一致性问题。

## 安装

官方提供了一个方便的脚本可以帮助我们快速完成 `docker` 的安装：

```bash
curl -fsSL https://get.docker.com | sudo bash

# 使用阿里云镜像
curl -fsSL https://get.docker.com | sudo bash -s --mirror Aliyun
```

默认情况下这个脚本会安装最新的 `Docker CLI`, `Docker Engine`, `Docker Buildx`, `Docker Compose`, `containerd`, `runc`。如果有其他安装需求可以参考 [Install Docker Engine](https://docs.docker.com/engine/install/) 。

此外，官方提供了 `docker` 的桌面软件可供使用，可参考 [Get Docker](https://docs.docker.com/get-started/get-docker/) 进行安装。

### 实用配置

- 网络代理/镜像加速

  国内使用 `docker` 可能出现一些网络问题，这里可以通过 网络代理 或者 [镜像加速](/hugo-blog/note/docker/docker_trick/#国内镜像加速) 来解决。

  配置网络代理，需要在 `daemon.json` 文件中添加 `proxies` 配置项：

  ```json
  {
    "proxies": {
      "http-proxy": "http://proxy.example.com:3128",
      "https-proxy": "https://proxy.example.com:3129",
      "no-proxy": "*.test.example.com,.example.org,127.0.0.0/8"
    }
  }
  ```

  `http-proxy` 和 `https-proxy` 配置为代理地址，`no-proxy` 配置哪些网站不走代理。

  完成修改后需要重启 `docker` 使配置生效：

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart docker
  ```

- 将用户加入Docker用户组

  将用户加入 `Docker` 用户组，避免每次都使用 `sudo` 执行命令。具体操作可以参考 [传送门](/hugo-blog/note/docker/docker_trick/#解决普通用户docker命令没有权限的问题)

<!-- 禁用 ipv6 -->

### Hello World

完成安装配置后，运行 `docker run hello-world` 命令（可能需要使用 `sudo`），看到类似以下结果说明安装成功了：

```bash
docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
17eec7bbc9d7: Pull complete
Digest: sha256:05813aedc15fb7b4d732e1be879d3252c1c9c25d885824f6295cab4538cb85cd
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## 基础概念

### 容器

容器（`container`）是 `docker` 中运行工作负载的最小单位，一个完整的应用可以由一个或多个容器构成：

- 容器包含完整的运行环境（在宿主机上运行所需要的所有资源）。
- 容器之间相互隔离，互不影响。
- 容器之间相互独立，它们可以独立的被管理。
- 容器可以以同样的方式方便地运行在开发、测试、生产等环境中。
- 容器运行在宿主机内核上，而非完整模拟的操作系统上。

容器可以是一个前端Web服务、一个后端API服务、一个数据库服务等等。

### 镜像

镜像（`image`）是容器运行的只读模板，它包含运行所需的所有文件，相当于容器的 `“源代码”`：

- 镜像是由多层文件系统组成的，每一层都是对镜像中文件的变动记录（增、删、改）。
- 镜像一旦创建就不可以再修改。我们只能通过在镜像顶部追加新的层创建新的镜像实现修改。

### 仓库

仓库（`registry`）是镜像存储和分发的地方，分为公共仓库和私有仓库。仓库为我们提供了一种统一管理镜像的方案，可以方便的实现不同机器间的镜像共享。

### 存储

默认情况下容器创建的文件都放在镜像只读层顶部的一个可写容器层上，这个可写容器层是不进行持久化的，容器销毁时这一层数据也会丢失。`docker` 存储就是为了解决数据持久化和容器间数据共享的问题。

容器支持一下挂载类型，让容器在可写容器层外存储数据：

| 类型                    | 特点                                              | 适用场景                                 |
| :---------------------- | :------------------------------------------------ | :--------------------------------------- |
| 卷（Volumes）           | 挂载 `docker` 管理的宿主机目录到容器中            | 持久化存储、容器间共享数据               |
| 绑定挂载（Bind Mounts） | 将宿主机任意目录 / 文件挂载到容器内               | 开发时实时同步代码、挂载宿主机配置文件   |
| tmpfs 挂载              | 数据只存在于宿主机内存中，容器停止即消失          | 临时存储敏感数据（如密码），避免写入磁盘 |
| 命名管道（Named pipes） | 将Windows 主机上的命名管道映射到 Windows 容器内部 | 针对Windows 平台设计的特殊存储挂载方式   |

### 网络

网络解决的是容器之间、容器与宿主机、容器与外部网络的通信问题，`docker` 启动后会自动创建默认网络。

核型类型：

| 类型                | 特点                                            | 用场景                                   |
| :------------------ | :---------------------------------------------- | :--------------------------------------- |
| 桥接网络（bridge）  | 默认网络，同一宿主机的容器可通过容 器名/IP 通信 | 单机多容器通信（如前端 + 后端 + 数据库） |
| 主机网络（host）    | 容器直接使用宿主机的网络栈，无网络隔离          | 追求网络性能、需要暴露大量端口的场景     |
| 覆盖网络（overlay） | 跨宿主机的容器通信，需配合 `Docker Swarm`       | 多机集群部署（如分布式应用）             |
| 无网络（none）      | 容器无网络接口，完全隔离                        | 无需网络的离线任务（如数据处理）         |

## 基础命令

虽然 `Docker CLI` 提供了大量命令来进行交互，实际使用中我们仅需要记忆一些常用的命令即可，其他命令可以需要时再查询 [参考手册](https://docs.docker.com/reference/) 即可。

> 所有命令中，容器ID/镜像ID 无需输入完整，输入前 `3-4` 位唯一字符即可。

### 镜像

```bash
# 拉取镜像
docker pull [选项] 镜像名:标签  # 例：docker pull nginx:1.25 、docker pull mysql

# 查看本地镜像
docker images  # 简写：docker image ls

# 删除本地镜像
docker rmi [选项] 镜像名:标签/镜像ID  # 例：docker rmi nginx:latest 、docker rmi -f 38299525e29e

# 清理无用镜像
docker image prune  # 按提示确认，加 -f 直接清理无需确认
```

### 容器

```bash
# 直接基于镜像创建并启动容器，-d 后台运行、-p 端口映射（宿主端口：容器端口） -v 挂载卷
docker run -d -p 宿主端口:容器端口 -v 卷名:容器内路径 --name 容器名 镜像名:标签 # docker run -d -p 3306:3306 -v mysql-data:/var/lib/mysql --name my-mysql mysql


# 查看运行中容器
docker ps
# 查看所有容器  包含运行中 + 已停止的所有容器
docker ps -a

# 启动已停止容器
docker start 容器名/容器ID
# 停止运行中容器 优雅停止（等待容器进程退出）
docker stop 容器名/容器ID

# 删除单个已停止容器
docker rm 容器名/容器ID
# 强制删除容器
docker rm -f 容器名/容器ID
# 清理所有已停止容器
docker container prune  # 加 -f 直接清理无需确认

# 进入容器内部 （优雅方式，不中断容器主进程）
docker exec -it 容器名/容器ID /bin/bash  # 大部分Linux容器用/bin/bash，轻量容器可用sh

# 查看容器日志
docker logs -f 容器名/容器ID  # -f 实时跟踪日志输出
```

### 其他

```bash
docker info  # 查看Docker系统信息（版本、镜像数、容器数、存储驱动等）

docker stats  # 实时查看所有运行中容器的资源占用（CPU、内存、磁盘IO、网络IO）
```

## Docker Hub

[Docker Hub](https://hub.docker.com/) 是 `Docker` 官方维护的公共镜像仓库，也是全球最大、最权威的 `Docker` 镜像平台，绝大多数常用软件的官方镜像都在此发布，是日常使用的首选。

1. 打开官网后，在顶部搜索框直接输入软件名称，例如 nginx、mysql、redis、python、ubuntu 等，即可找到对应的镜像。搜索结果中，带有 `「Official Image」` 标识的，是软件官方维护的正版镜像，安全性和稳定性最高，优先选择。
2. 点击镜像进入详情页，详情页会直接给出标准的 `docker pull` 命令，直接复制到终端执行即可拉取镜像。此外，在详情页面还可以看到：
   - 支持的版本标签（`Tag`），如 `latest`、`8.0``、alpine` 等，建议指定具体版本，避免使用默认 `latest` 导致版本不可控。
   - 镜像的使用文档、环境变量配置、端口说明、数据卷挂载等官方指引。
   - 镜像的更新日志、架构支持（`amd64`、`arm64` 等）。

## 参考资料

[Docker 官方文档](https://docs.docker.com/)  
[Docker Hub 官网](https://hub.docker.com/)
