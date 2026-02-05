---
date: "2026-02-03T14:28:44+08:00"
title: "Docker -- docker compose"
tags: ["Docker", "docker compose"]
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

## 常用命令

`docker compose` 提供了完整方便的应用管理命令：

- `docker compose up` 启动应用
- `docker compose down` 停止并销毁应用
  默认情况下会停止并删除容器及默认网络，可以添加参数 `-v` 或者 `--volumes` 删除命名数据卷（慎用），添加参数 `--rmi all` 同时删除构建所使用到的镜像。
- `docker compose stop` 停止应用。
  所有资源都会保留，可以使用 `docker compose -d` 快速恢复。
- `docker compose restart` 重启应用。
  默认重启应用中所有服务对应的容器，可以通过 `docker compose restart <服务名>` 重启指定的服务。
- `docker compose ps` 查看容器（服务）运行状态。
- `docker compose log` 查看容器（服务）日志。
  默认查看整个应用的日志，可使用 `docker compose log <服务名>` 查看指定服务的日志。配合 `-f` 持续追踪查看实时日志输出。
- `docker compose exec <服务名> <命令>` 进入容器执行命令。
  类似 `docker exec`，使用示例 `docker-compose exec db bash`。

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

在启动应用的时候，我们通常有 服务器使用指定的配置文件、指定的数据文件、持久化产生的数据 这类需求，这就需要使用数据卷绑定功能。`docker compose` 支持 `主机绑定挂载` 和 `命名卷` 方式。

### 主机绑定挂载

主机绑定挂载允许我们将本地目录映射到容器内的指定目录，通过 `services` 元素下的 `volumes` 属性进行设置。

这里我们以为修改 `nginx` 的主页为例子进行演示：

1. 在之前使用的 `compose` 文件下创建 `html` 文件夹，添加 `index.html`。

```bash
mkdir html
touch index.html
```

在 `index.html` 中添加以下内容：

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <h1>Hello World</h1>
  </body>
</html>
```

2. 修改 `compose.yaml` 文件：

```yaml
# compose.yaml
name: "web"
services:
  nginx:
    image: "nginx:1.28.1-alpine-slim"
    ports:
      - "9999:80"
    # 添加 volumes 属性
    volumes:
      # 绑定格式为  本地路径:容器内路径
      - "./html:/usr/share/nginx/html/"
```

3. 更新应用进行测试：

```bash
docker compose up -d
[+] Running 2/2
✔ Network web_default    Created                                                                                                                                          0.0s
✔ Container web-nginx-1  Started                                                                                                                                          0.1s
```

```bash
curl http://localhost:9999

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <h1>Hello World</h1>
</body>
</html>%
```

可以看到返回内容为我们准备的 `html` 文件，说明路径已经映射到容器内容。

### 命名卷

命名卷的创建是 `docker compose` 进行管理，我们只需要在 `compose` 文件中进行声明、绑定即可使用。

声明命名卷需要使用顶级元素 `volumes`:

```yaml
volumes:
  nginx-data:
```

绑定方法同主机绑定挂载一样，仅需将本地路径替换成命名卷名即可：

```yaml
services:
  backend:
    image: example/database
    volumes:
      - db-data:/etc/data

  backup:
    image: backup-service
    volumes:
      - db-data:/var/lib/backup/data

volumes:
  db-data:
```

## 环境变量

`docker compose` 支持为服务（容器）添加环境变量。通过在 `service` 的 `environment` 添加配置实现，配置格式如下：

```yaml
services:
  db: # 服务名：db
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
```

或者：

```yaml
services:
  db: # 服务名：db
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=123456
```

## 重启策略

通过对 `service` 添加属性 `restart`，可以控制服务（容器）退出后的重启策略，目前支持一下配置：

- `no` 默认配置，不进行重启。
- `always` 只要容器退出就进行重启，知道容器被移除。即使是手动停止容器，在 `docker` 守护进程重启（如：服务器重启时）后任然会进行重启。
- `on-failure[:max-retries]` 异常退出的时候进行重启，异常退出的判定基于退出时的状体码（exit code）。可以添加最大尝试次数（`max-retries`） 限制。
- `unless-stopped` 只要容器停止就进行重启，除非主动停止或者被移除。

使用示例：

```yaml
services:
  test-app:
    build: .
    ports:
      - "8081:8080"
    restart: unless-stopped # 测试环境：自动重启，手动停止后不恢复
  # 缓存服务
  redis:
    image: redis:7.0
    volumes:
      - redis-data:/data
    restart: always # 缓存：核心依赖，无条件重启
    # 数据同步脚本：异常退出重启，最多重试3次
  data-sync:
    build: ./sync
    restart: on-failure:3 # 非0退出重启，最多3次，避免致命错误无限重启
  # 日志收集服务：任意异常退出都重启，不限制次数
  log-collect:
    image: logstash:8.0
    restart: on-failure # 仅异常重启，无次数限制
```

## 启动顺序

通过服务的 `depends_on` 属性，我们可以描述服务的依赖关系，从而控制服务的启动、停止顺序等。

`depends_on` 属性的内容是一个数组，数组中的元素是其他服务名。使用方法如下：

```yaml
version: "3.8" # 兼容所有 3.x 版本，2.x 也支持
services:
  # 业务服务：依赖 db 和 redis，需后启动
  app:
    image: my-app:latest
    ports:
      - "8080:8080"
    depends_on:
      - db # 依赖数据库服务
      - redis # 依赖缓存服务

  # 数据库服务：无依赖，最先启动
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
    volumes:
      - mysql-data:/var/lib/mysql

  # 缓存服务：无依赖，与 db 并行启动（无相互依赖时）
  redis:
    image: redis:7.0
    volumes:
      - redis-data:/data

volumes:
  mysql-data:
  redis-data:
```

启动顺序说明：

1.  无依赖的 `db` 和 `redis` 会并行启动（提升启动效率）；
2.  等待 `db` 和 `redis` 均启动（容器状态为 `running`）后，再启动 `app` 服务；
3.  若依赖服务启动失败，当前服务会直接终止启动，避免无效运行。

> 有时候即使容器状态为 `running` 时，但服务还没有就绪。此时启动依赖服务可能造成一些异常报错。`Compose 3+` 后 `denpends_on` 提供了健康检查功能（healthcheck）实现更加精细的启动顺序控制。详细用法可以参考[官方文档 -- depends_on](https://docs.docker.com/reference/compose-file/services/#depends_on)。

## 待续。。。

<!-- todo 补充 资源限制-->

## 参考资料

[Docker 官方文档](https://docs.docker.com/)
