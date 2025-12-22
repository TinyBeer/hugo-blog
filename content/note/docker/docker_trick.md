---
date: '2025-12-22T14:58:35+08:00'
title: 'Docker -- 实用技巧总结 Linux'
tags: ["Docker", "Linux"]
categories: "笔记"
# description: "Desc Text."
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

本文总结了日常 Linux 下使用 Docker 的一些技巧。
<!--more-->

## Linux 一键安装脚本
> 如果出现  
> `curl: (35) OpenSSL SSL_connect: Connection reset by peer in connection to download.docker.com:443`  
> 可以尝试安装ca认证工具后重试  
> `sudo apt-get update && sudo apt-get install ca-certificates`


```bash
curl -fsSL https://get.docker.com | bash
# 使用阿里云镜像资源
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

## Amazon Linux 中安装 Docker
一键安装脚本目前暂不支持在 Amazon Linux 中使用，这里提供了手动安装的方法。
```bash
sudo dnf update -y
sudo dnf install docker -y
sudo systemctl start docker
sudo systemctl enable docker
```

## 解决普通用户docker命令没有权限的问题
将当前用户加入docker用户组
> 如果没有docker与用户组  
> 查看用户组是否存在 sudo cat /etc/group | grep docker  
> 添加用户组 sudo groupadd docker  

```bash
# 加入docker用户组  需要重启
sudo usermod -aG docker $USER
# 刷新docker用户组 当前会话生效
newgrp docker
```

## 国内镜像加速
```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json << EOF
{
    "registry-mirrors": ["https://register.librax.org/","https://docker.gh-proxy.com/"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 批量镜像文件处理
* 打包成一个文件
    ```bash
    # 直接使用-q会丢失tag,成为虚悬镜像
    docker save $(docker images --format "{{.Repository}}:{{.Tag}}") -o all_images.tar
    ```
* 分别打包
    ```bash
    #!/bin/bash
    for image in $(docker images -q); do
        tag=$(docker inspect --format='{{join .RepoTags ":"}}' $image)
        if [ -n "$tag" ]; then
            filename=$(echo $tag | tr ':' '_').tar
            docker save -o $filename $image
        fi
    done
    ```
* 批量加载本地镜像tar包
    ```bash
    for image_file in *.tar; do docker load -i "$image_file"; done
    ```