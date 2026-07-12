---
date: "2024-05-30T15:40:52+08:00"
title: "Go模块管理速查手册.md"
tags: ["Golang", "mod", "workspace"]
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

本文将对`go mod`和`go work`两个依赖管理工具进行简单的说明。未能解释清楚的地方可自行查阅官方文档。<!--more-->

## module

`go module`是 Go 1.11 版本之后官方推出的版本管理工具，并且从 Go 1.13 版本开始，`go module`成为了 Go 语言默认的依赖管理工具。

### 重要环境变量

#### GO111MODULE

`GO111MODULE`是用于控制`go module`工作模式的环境变量，它有三个可选值：

1. `off` 禁用，编译时从`GOPATH`和`vendor`中查找依赖包。
2. `on` 启用，编译时只根据`go.mod`管理依赖。
3. `auto` 自动，当项目不在`$GOPATH/src`下，并且项目根目录有`go.mod`文件时，开启模块支持。

#### GOPROXY

`GOPROXY`用于配置下载依赖的代理地址，默认值为`https://proxy.golang.org`，但对于国内用户不是很友好。
可以使用其他镜像源替代，如`goproxy.cn`

```
go env -w GOPROXY=https://goproxy.cn,direct
```

此外还有

```
https://mirrors.aliyun.com/goproxy

https://goproxy.io

https://gocenter.io

...
```

`GOPROXY`中的`direct`表示对于不在代理中的包，直接从源站拉取，而不是返回错误。

#### GOPRIVATE / GONOSUMDB

当项目依赖私有仓库时，需要配置以下环境变量：

```
# 私有仓库不走代理，直接从源站拉取
go env -w GOPRIVATE=github.com/yourcompany/*

# 私有仓库不进行 sum 校验（私有仓库在 sumdb 中没有记录）
go env -w GONOSUMDB=github.com/yourcompany/*
```

常见的配置组合：

```
GOPRIVATE=github.com/yourcompany/*,gitee.com/yourcompany/*
GONOPROXY=github.com/yourcompany/*,gitee.com/yourcompany/*
GONOSUMDB=github.com/yourcompany/*,gitee.com/yourcompany/*
```

### `go mod`常用命令

- `go mod init xxx` 初始化当前文件夹，创建 go.mod 文件 xxx 为希望的 module 名
- `go mod tidy` 增加缺少的 module，删除无用的 module
- `go mod vendor` 将依赖复制到 vendor 下
- 其他命令可使用`go mod help`自行查看

### `go.mod`文件

例子

```
module github.com/Q1mi/studygo/blogger

go 1.12

require (
	github.com/DeanThompson/ginpprof v0.0.0-20190408063150-3be636683586
	github.com/gin-gonic/gin v1.4.0
	github.com/go-sql-driver/mysql v1.4.1
	github.com/jmoiron/sqlx v1.2.0
	github.com/satori/go.uuid v1.2.0
	google.golang.org/appengine v1.6.1 // indirect
)

replace (
	github.com/gin-gonic/gin v1.4.0 => ../gin-gonic/gin
)
```

- module 用于定义包名，或者叫模块名
- require 用于指定依赖包
- replace 用于重定向，支持本地路径和版本替换

```go
// 重定向到本地路径（开发调试常用）
replace github.com/gin-gonic/gin v1.4.0 => ../gin-gonic/gin

// 版本替换（如修复了某个依赖的 bug，等待上游合并）
replace golang.org/x/crypto v0.0.0 => github.com/golang/crypto v0.0.1-20230101

// 替换到 fork 仓库
replace github.com/original/pkg => github.com/yourfork/pkg v1.0.0
```

- indirect 表示间接引用

#### `go.sum`文件

`go.sum`是依赖的校验文件，记录了每个依赖包的哈希值，用于确保依赖的完整性和一致性。该文件应该提交到版本控制系统中。

```
github.com/DeanThompson/ginpprof v0.0.0-20190408063150-3be636683586 h1:...
github.com/DeanThompson/ginpprof v0.0.0-20190408063150-3be636683586/go.mod h1:...
```

- 第一列是模块路径和版本
- `h1:` 开头的是模块 zip 包的 SHA-256 哈希
- `/go.mod` 后缀的是`go.mod`文件本身的哈希

> 运行`go mod tidy`时会自动更新`go.sum`，一般不需要手动编辑。

#### 大版本路径规则

Go 模块从 `v2` 开始需要在模块路径中添加版本后缀：

```go
// v1.x.x - 正常路径
require github.com/example/mymodule v1.2.3

// v2.x.x - 需要 /v2 后缀
require github.com/example/mymodule/v2 v2.0.0

// go.mod 中也需要声明
module github.com/example/mymodule/v2
```

这意味着升大版本时需要：

1. 在`go.mod`中修改 module 路径，加上`/v2`后缀
2. 代码中所有 import 路径也要同步修改
3. 创建对应的 Git tag（如`v2.0.0`）

### `go get`命令

在项目中执行`go get`命令可以下载依赖包，并且还可以指定下载的版本。

1. `go get -u` 将会升级到最新的次要版本或者修订版本（x.y.z，z 是修订版本号，y 是次要版本号）
2. `go get -u=patch` 将会升级到最新的修订版本
3. `go get package@version` 将会升级到指定的版本号 version

   下载所有依赖可以使用`go mod download`命令

### 其他操作

- 格式化`go.mod`文件
  在手动修改`go.mod`文件后，使用`go mod edit -fmt`格式化`go.mod`文件
- 命令行重定向包
  `go mod edit -replace=golang.org/x/crypto@v0.0.0=github.com/golang/crypto@latest`
- 清理`module`缓存
  `go clean -modcache`
- 查看可下载的包版本
  `go list -m -versions xxx`
- 校验依赖完整性，检查下载的模块是否被篡改
  `go mod verify`
- 查看模块依赖关系图，排查版本冲突时很有用
  `go mod graph`
- 查看某个包为什么被引入，清理无用依赖时使用
  `go mod why -m github.com/example/pkg`
- 使用最新包 **慎用**，`require` 中不支持 `latest` 语法，需指定具体版本

```
# 错误写法 - latest 在 go.mod 中无效
require (
	github.com/DeanThompson/ginpprof latest
	github.com/gin-gonic/gin latest
)

# 正确写法 - 使用 go get 获取最新版本
# go get github.com/DeanThompson/ginpprof@latest
```

## Workspaces

在`go 1.18`后，使用 workspaces 支持开发者同时在多个 module 中间进行编码（不需要修改 go.mod 文件），可以方便的进行编译运行等工作。常用于`common`组件开发，如日志库、错误库等等。

### 常用命令

- `go work` 或 `go help work` 查看帮助文档
- `go work init` 初始化 workspaces 文件，创建一个`go.work`文件，并将添加指定文件夹中的 module。`go work init ./hello`
- `go work use` 添加新的 module 到`go.work`文件中。`go work use ./example`
- `go work sync` 同步 workspace 内各 module 的依赖版本，将所有 module 的依赖统一到兼容版本
- `go work edit` 命令行编辑`go.work`文件。`go work edit -use=./newmodule`

### 实际操作

1. 创建 `workspace` 工作目录

   ```
   $ mkdir workspace
   $ cd workspace
   ```

2. 创建`hello` module

   ```
   $ mkdir hello
   $ cd hello
   $ go mod init example.com/hello
   go: creating new go.mod: module example.com/hello
   ```

   在`hello`文件夹下添加`hello.go`

   ```golang
   package main

   import (
       "fmt"
   )

   func main() {
       fmt.Println("hello")
   }
   ```

   回到`workspace`目录下，执行

   ```
   $ go work init ./hello
   ```

   产生一个`go.work`文件，内容如下。

   ```
   go 1.18

   use ./hello
   ```

3. 添加依赖`example`
   - 在 workspace 目录下创建 example 文件夹，并初始化 go mod

     ```sh
     mkdir example
     cd example
     go mod init example
     ```

     实际开发中这里的 go mod 应该指向代码仓库，如`github.com/xxx/xxx`

   - 添加新`module`，回到 workspace 目录执行

     ```
     go work use ./example
     ```

     此时，`go.work`文件会增加`example`相关信息。

     ```
     go 1.18

     use (
         ./example
         ./hello
     )
     ```

4. 修改本地代码进行测试
   在`workspace/example/`目录下创建一个新文件`stringutil/test.go`

   ```
   package stringutil

   func ToUpper(s string) string {
       return "prefix " + s
   }
   ```

   修改`workspace/hello/hello.go`代码如下：

   ```
   package main

   import (
       "fmt"

       "example/stringutil"
   )

   func main() {
       fmt.Println(stringutil.ToUpper("hello"))
   }
   ```

   运行测试

   ```
   $ go run example.com/hello
   HELLO
   ```

5. 最终的文件结构如下：

   ```sh
   │  go.work
   │
   ├─example
   │  │  go.mod
   │  │
   │  └─stringutil
   │          test.go
   │
   └─hello
           go.mod
           go.sum
           hello.go
   ```

   我们开发 example 中的公共代码，并在 hello 中进行测试。

## 参考资料

- [Go 语言之依赖管理](https://www.liwenzhou.com/posts/Go/go_dependency/)
- [Go mod 常用与高级操作](https://www.cnblogs.com/span-phoenix/p/15311316.html)
- [Tutorial: Getting started with multi-module workspaces](https://go.dev/doc/tutorial/workspaces)
