---
date: "2026-01-31T22:36:09+08:00"
title: "Docker -- 镜像构建"
tags: [“Docker”]
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

`Docker` 镜像构建是 `Docker` 容器化的核心与前提。`docker build` 是从构建上下文创建镜像的核心命令。

<!--more-->

它通过读取指定目录下的 `Dockerfile` 构建规则，自动执行镜像分层构建、文件复制、命令执行等操作，最终生成可重复使用、可移植的 `Docker` 镜像，是 `Docker` 镜像定制的标准方式。

`docker build` 使用 `C/S` 架构实现：
客户端： `Buildx` 提供管理和执行构建的接口。
服务端： `BuildKit` 实际执行构建过程。

## 构建一个镜像

`Docker` 通过读取 `Dockerfile` 中的命令来控制镜像的构建过程。通常我们将其中存在在一个名为 `Dockerfile` 的文件中，这个文件没有拓展名。通过 `docker build .` 命令执行构建时，`docker` 会在当前文件夹下寻找 `Dockerfile` 指导构建。当然我们也可通过 `-f` 参数指定 `Dockerfile`。

> 这里的 `Dockerfile` 是一段包含用于构建镜像的命令的一段文本，大多数情况下我们使用一个文件存放它。

这里对常用的指令进行说明，完整语法说明可以参考 [Dockerfile reference](https://docs.docker.com/reference/dockerfile)。

| 指令                   | 说明                                                                                                                               |
| :--------------------- | :--------------------------------------------------------------------------------------------------------------------------------- |
| FROM `<image>`         | 定义一个基础镜像。                                                                                                                 |
| ENV                    | 设置环境变量                                                                                                                       |
| RUN `<command>`        | 在当前镜像之上的新层中执行任何命令并提交结果。RUN 还有一个 shell 形式，用于运行命令。                                              |
| WORKDIR `<directory>`  | 设置 `Dockerfile` 中 `RUN`、`CMD`、`ENTRYPOINT`、`COPY` 和 `ADD` 指令的工作目录。                                                  |
| COPY `<src>` `<dest>`  | 从 `<src>` 复制新的文件或目录，并将它们添加到路径 `<dest>` 的容器文件系统中。                                                      |
| ADD `<src>` `<dest>`   | 同样是复制文件，但支持解压压缩包 / 拉取远程文件。                                                                                  |
| CMD `<command>`        | 允许您定义在基于此镜像启动容器后运行的默认程序。每个 `Dockerfile` 只有一个 `CMD`，当存在多个 `CMD` 实例时，仅最后一个 `CMD` 生效。 |
| ENTRYPOINT `<command>` | 同样是指定容器入口命令，但不可被覆盖。可以配置 `CMD` 进行传参。                                                                    |
| EXPOSE `<port>`        | 声明容器监听的端口（仅声明，不实际映射） 参数可以是 `端口/协议`                                                                    |

下面通过一个基于 `Go` 语言的、简单的 `HTTP` 服务演示如何构建镜像：

- 源代码：

创建项目：

```bash
mkdir hello_world
cd hello_world
go mod init hello_world
touch main.go

# 项目结构如下：
tree
.
├── go.mod
├── go.sum
└── main.go

```

在 `main.go` 中添加代码：

```golang
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/hello", func(c *gin.Context) {
		c.String(http.StatusOK, "hello world")
	})

	r.Run(":9999")
}

```

测试：

```bash
# 下载依赖
go mod tidy
# 启动服务
go run .

# 另启一个终端
curl localhost:9999/hello
```

如果一切正常可以看到终端打印出字符串 `hello world`

- 编写 `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25-alpine

# 设置构建时环境变量：强制Go编译为Linux平台可执行文件（避免本地系统影响）
ENV CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64\
    # 配置国内代理，一行写全，用\分隔
    GOPROXY=https://goproxy.cn,direct

# 设置工作目录（后续指令默认在该目录执行，自动创建）
WORKDIR /app

# 复制项目所有源码到工作目录
COPY . .

# 下载项目所有依赖（提前下载，复用缓存）
RUN go mod tidy

# 编译Go代码：生成可执行文件，命名为app（-o 指定输出文件名）
# -ldflags "-s -w"：精简可执行文件（去除调试信息、符号表），减小文件体积
RUN go build -ldflags "-s -w" -o hello_world main.go

# 声明容器监听的端口（与Go应用的9999端口一致，仅声明，不实际映射）
EXPOSE 9999/tcp

# 容器启动时执行的命令：运行可执行文件
CMD ["./hello_world"]
```

> [!important] `FROM golang:1.25-alpine` 指明使用的基础镜像，其中包含 `go1.25` 这里的版本一定要满足 `go.mod` 中需求的版本，否则会报错。

第一行中的 `# syntax=docker/dockerfile:1` 是语法解析器指令（其他以 `#` 开头的都是注释），尽管该指令是可选的，但它会告知 `Docker` 构建器在解析 `Dockerfile` 时使用何种语法，并且能让已启用 `BuildKit` 的旧版 `Docker`，在开始构建前加载并使用指定的 `Dockerfile` 前端解析器。解析器指令必须出现在 `Dockerfile` 中所有其他注释、空白符或 `Dockerfile` 指令之前，且应当作为 `Dockerfile` 的第一行。

构建镜像：

执行 `ocker build -t hello_world:v1.0 .` 开始构建：

- `-t hello_world:v1.0` 为构建的镜像指定 `tag`。
- `.` 表示使用当前目录作为构建上下文。

可以看到 `docker build` 将完整的构建过程打印了出来：

```bash
docker build -t hello_world:v1.0 .
DEPRECATED: The legacy builder is deprecated and will be removed in a future release.
            Install the buildx component to build images with BuildKit:
            https://docs.docker.com/go/buildx/

Sending build context to Docker daemon  14.34kB
Step 1/8 : FROM golang:1.25-alpine
 ---> 98e6cffc31cc
Step 2/8 : ENV CGO_ENABLED=0     GOOS=linux     GOARCH=amd64    GOPROXY=https://goproxy.cn,direct
 ---> Running in abb049a6487b
 ---> Removed intermediate container abb049a6487b
 ---> 98049202b3af
Step 3/8 : WORKDIR /app
 ---> Running in 445c9dd4e403
 ---> Removed intermediate container 445c9dd4e403
...
```

打印完成后，执行 `docker images` 可以看到一个新的镜像 `hello_world:v1.0` 被创建了出来。

测试：

```bash
# 后台启动 hello_world:v1.0 镜像 容器名称为 hello_world 映射 9999 端口到 宿主机 9999 端口
docker run -d --name hello_world -p 9999:9999 hello_world:v1.0

# 测试访问
curl localhost:9999/hello
hello world
```

## 多阶段构建

刚才我们完成了 `hello_world` 镜像的构建和运行测试。虽然我们的操作是成功的，但是我们的镜像中包含了运行所不需要的 `go` 环境，依赖包，源代码等内容，这使得我们的镜像不够精巧。事实上运行这个应用，我们只需要一个包含可执行文件的镜像即可。这里有两种处理方案。其一，在其他地方生产可执行文件，然后在进行镜像构建。其二，便是 `Docker` 的多阶段构建（`Multi-stage builds`）。

开启多阶段构建，我们只需修改之前的 `Dockerfile` 即可实现，修改后的结果如下：

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.25-alpine AS builder


# 设置构建时环境变量：强制Go编译为Linux平台可执行文件（避免本地系统影响）
ENV CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64\
    # 配置国内代理，一行写全，用\分隔
    GOPROXY=https://goproxy.cn,direct

# 设置工作目录（后续指令默认在该目录执行，自动创建）
WORKDIR /app

# 复制项目所有源码到工作目录
COPY . .

# 下载项目所有依赖（提前下载，复用缓存）
RUN go mod tidy

# 编译Go代码：生成可执行文件，命名为app（-o 指定输出文件名）
# -ldflags "-s -w"：精简可执行文件（去除调试信息、符号表），减小文件体积
RUN go build -ldflags "-s -w" -o hello_world main.go

# 使用没有golang环境的基础镜像
FROM alpine:3.19

WORKDIR /app

# 从一阶段构建中拷贝可执行文件
COPY --from=builder /app/hello_world .

# 声明容器监听的端口（与Go应用的9999端口一致，仅声明，不实际映射）
EXPOSE 9999/tcp

# 容器启动时执行的命令：运行可执行文件
CMD ["./hello_world"]
```

变化内容如下：

- `FROM golang:1.25-alpine AS builder`: 使用 `AS` 为构建指定别名，方便后续阶段使用其中产物。
- `FROM alpine:3.19`： 使用不含 `go` 环境的基础镜像
- `COPY --from=builder /app/hello_world .`： 通过 `--from` 指定从之前的构建中拷贝文件

执行构建：

```bash
# 为了和之前的构建区别开  这里使用了 tag v2.0 进行区分
docker build -t hello_world:v2.0 .
```

查看结果：

```bash
docker images

IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:3.19          6baf43584bcb       12.5MB         3.51MB
golang:1.25-alpine   98e6cffc31cc        324MB         64.5MB
hello_world:v1.0     3fde28a84e3b        894MB          186MB
hello_world:v2.0     081865bed03e       31.4MB         8.48MB
```

可以看到 `v2.0` 版本在磁盘占用上有了明显的优化。

## 构建缓存

`Build Cache` 是 `Docker` 为提升 `docker build` 构建效率设计的核心分层缓存机制，核心原理是对 `Dockerfile` 中每一条指令的执行结果生成只读镜像层并缓存，后续构建时若指令及关联的构建上下文内容未发生变化，直接复用已有缓存层，无需重新执行指令，可大幅减少重复构建的时间（尤其适合频繁修改代码的开发场景）。

`Dockerfile` 中每一条有效指令（如 `FROM`/`RUN`/`COPY` 等）执行后都会生成一个独立的只读镜像层，`Build Cache` 正是基于这些镜像层实现缓存复用，核心判定逻辑为：

1. `Dockerfile` 指令会从第一条开始逐行检查是否可复用缓存；
2. 对每一条指令，对比「当前构建的指令内容 + 关联的构建上下文文件」与「缓存中对应层的指令 + 文件」是否完全一致；
3. 若一致则复用缓存层，直接跳过该指令的执行；若不一致则失效当前指令及后续所有指令的缓存，从当前指令开始重新构建所有后续层（缓存失效具有「传递性」，某一层失效后，后续所有层均无法复用缓存）；
4. 构建完成后，新生成的镜像层会替代原有缓存层，更新到本地 Build Cache 中。

简而言之，缓存的复用从第一条指令开始，一旦某一行指令缓存失效，后续所有行都将重新构建。

> [!important] `RUN` 命令匹配缓存是基于命令参数的完全匹配的，需要避免使用 `RUN git clone <最新分支>` 这类无实际可控的缓存价值的命令。

### 优化思路

基于缓存判定规则我们可以从以下方向进行优化：

1. 指令尽量按照稳定度从高到低组织
   高频不变的指令放前面，高频变化的指令放后面。、
2. 精准拷贝文件，避免引入无关变化文件
   在使用 `COPY` 和 `ADD` 命令时，避免使用 `COPY . .` 进行全部拷贝，而是精确的控制拷贝哪些文件。或合理的使用 [`.dockerignore`](https://docs.docker.com/build/concepts/context/#dockerignore-files) 过滤掉不会使用到的 文件/文件夹。
3. 合并 RUN 指令，减少镜像分层与缓存判定次数
   将多条相关的 RUN 指令通过 && 合并为一条，减少镜像分层数量，同时降低缓存判定的次数（分层越少，缓存复用的效率越高）。这一条需要更具具体情况使用。

## 参考资料

[Docker 官方文档](https://docs.docker.com/)
