---
date: "2026-02-26T09:49:25+08:00"
title: "takama/daemon -- 以守护进程方式运行服务"
tags: ["Golang", "takama/daemon"]
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

`takama/daemon` 是一个用于将 `Go` 程序封装成系统守护进程（服务）的库，支持 `macOS`、`FreeBSD`、`Linux（SystemD/Upstart/SystemV）` 和 `Windows` 系统。

<!--more-->

## 核心概念

`takama/daemon` 提供了多种 `Daemon` 类型来实现对不同系统的支持：

| 系统    | Daemon类型                         | 底层依赖                    |
| :------ | :--------------------------------- | :-------------------------- |
| Linux   | SystemDaemon                       | SystemD > Upstart > SystemV |
| Windows | SystemDaemon                       | Windows Service API         |
| FreeBSD | SystemDaemon                       | BSD 服务管理                |
| macOS   | UserAgent/GlobalAgent/GlobalDaemon | Launchd                     |

- `SystemDaemon`：适用于 FreeBSD/Linux/Windows 的系统级守护进程（需 `root` / 管理员 权限）；
- `UserAgent/GlobalAgent/GlobalDaemon`：仅适用于 macOS 的用户级 / 全局级守护进程。

所有 `Daemon` 类型都实现了一下方法来管理服务：

- `Install()`：安装服务；
- `Remove()`：卸载服务；
- `Start()`：启动服务；
- `Stop()`：停止服务；
- `Status()`：查询服务状态。

## 使用方法

### 安装

```bash
go get github.com/takama/daemon
```

### 基础模板

```golang
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/takama/daemon"
)

const (
	name        = "myservice"  // 服务名称
	description = "My Service" // 服务描述
)

// 依赖的服务 如果无依赖则不填写
var dependencies = []string{"dummy.service"}

var stdlog, errlog *log.Logger

// Service 潜入 daemon 实体
type Service struct {
	daemon.Daemon
}

// Manage 通过 daemon 的方法管理服务
func (service *Service) Manage() (string, error) {

	usage := "Usage: myservice install | remove | start | stop | status"

	// 根据接收到的参数管理服务
	if len(os.Args) > 1 {
		command := os.Args[1]
		switch command {
		case "install":
			return service.Install()
		case "remove":
			return service.Remove()
		case "start":
			return service.Start()
		case "stop":
			return service.Stop()
		case "status":
			return service.Status()
		default:
			return usage, nil
		}
	}

	// 使用带缓存的通道接收信号  避免丢失信号
	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt, syscall.SIGTERM)

	// 启动协程运行业务逻辑
	go func() {
		for {
			//...
		}
	}()

	killSignal := <-interrupt
	stdlog.Println("Got signal:", killSignal)
	// 优雅的关闭服务
	// ...
	if killSignal == os.Interrupt {
		return "Daemon was interruped by system signal", nil
	}
	return "Daemon was killed", nil
}

func init() {
	stdlog = log.New(os.Stdout, "", log.Ldate|log.Ltime)
	errlog = log.New(os.Stderr, "", log.Ldate|log.Ltime)
}

func main() {
	// 使用 SystemDaemon  如果使用 macos 请根据需求替换为 UserAgent/GlobalAgent/GlobalDaemon
	srv, err := daemon.New(name, description, daemon.SystemDaemon, dependencies...)
	if err != nil {
		errlog.Println("Error: ", err)
		os.Exit(1)
	}
	service := &Service{srv}
	status, err := service.Manage()
	if err != nil {
		errlog.Println(status, "\nError: ", err)
		os.Exit(1)
	}
	fmt.Println(status)
}

```

### 实例演示

这里使用 `gin` 框架运行一个 `web` 服务，完整代码如下：

```golang
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"

	"github.com/gin-gonic/gin"
	"github.com/takama/daemon"
)

const (
	name        = "myservice"  // 服务名称
	description = "My Service" // 服务描述
	port        = ":8999"      // 服务监听端口
)

// 依赖的服务 如果无依赖则不填写
var dependencies = []string{}

var stdlog, errlog *log.Logger

// Service 潜入 daemon 实体
type Service struct {
	daemon.Daemon
}

// Manage 通过 daemon 的方法管理服务
func (service *Service) Manage() (string, error) {

	usage := "Usage: myservice install | remove | start | stop | status"

	// 根据接收到的参数管理服务
	if len(os.Args) > 1 {
		command := os.Args[1]
		switch command {
		case "install":
			return service.Install()
		case "remove":
			return service.Remove()
		case "start":
			return service.Start()
		case "stop":
			return service.Stop()
		case "status":
			return service.Status()
		default:
			return usage, nil
		}
	}

	// 使用带缓存的通道接收信号  避免丢失信号
	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt, syscall.SIGTERM)

	// 启动协程运行业务逻辑
	go service.run()

	killSignal := <-interrupt
	stdlog.Println("Got signal:", killSignal)
	// 优雅的关闭服务
	if killSignal == os.Interrupt {
		return "Daemon was interruped by system signal", nil
	}
	return "Daemon was killed", nil
}

func (service *Service) run() {
	r := gin.Default()

	r.GET("/hello", func(ctx *gin.Context) {
		ctx.String(http.StatusOK, "hello world!")
	})

	err := r.Run(port)
	if err != nil {
		errlog.Println("Error: ", err)
		os.Exit(1)
	}
}

func init() {
	stdlog = log.New(os.Stdout, "", log.Ldate|log.Ltime)
	errlog = log.New(os.Stderr, "", log.Ldate|log.Ltime)
}

func main() {
	// 使用 SystemDaemon  如果使用 macos 请根据需求替换为 UserAgent/GlobalAgent/GlobalDaemon
	srv, err := daemon.New(name, description, daemon.SystemDaemon, dependencies...)
	if err != nil {
		errlog.Println("Error: ", err)
		os.Exit(1)
	}
	service := &Service{srv}
	status, err := service.Manage()
	if err != nil {
		errlog.Println(status, "\nError: ", err)
		os.Exit(1)
	}
	fmt.Println(status)
}
```

```bash
# 构建
go build -o myservice .

# 安装
sudo ./myservice install
Install My Service:                                     [  OK  ]

# 查看
sudo ./myservice status
Service is stopped

# 运行
sudo ./myservice start
Starting My Service:                                    [  OK  ]

# 测试
curl http://localhost:8999/hello
hello world!%

# 停止
sudo ./myservice stop
Stopping My Service:                                    [  OK  ]

# 移除
sudo ./myservice remove
Removing My Service:                                    [  OK  ]
```

### 构建约束

对于不同系统的适配，除了使用 `runtime.GOOS` 进行判断外，还可以使用 `Go语言` 的 构建约束。通过 `Go语言` 的 `// +build` 构建约束（Build Tags）来实现不同系统下 `takama/daemon` 的差异化构建，这种方式比运行时判断更轻量，能在编译阶段就为不同系统生成适配的代码。

我们创建四个文件 `daemon_darwin.go`、`daemon_freebsd.go`、`daemon_linux.go`、`daemon_windows.go` 用来定义不同系统使用的 `daemon` 类型。

```golang
//go:build darwin
// +build darwin

package main

import "github.com/takama/daemon"

// 用户级  无需root权限  也可以替换成 GlobalDaemon（系统级）
var MyDaemon = daemon.UserAgent

```

```golang
//go:build freebsd
// +build freebsd

package main

import "github.com/takama/daemon"

var MyDaemon = daemon.SystemDaemon
```

```golang
//go:build linux
// +build linux

package main

import "github.com/takama/daemon"

var MyDaemon = daemon.SystemDaemon

```

```golang
//go:build windows
// +build windows

package main

import "github.com/takama/daemon"

var MyDaemon = daemon.SystemDaemon

```

然后替换 `main.go` 中的 `daemon` 类型：

```golang
...
	// 使用 SystemDaemon  如果使用 macos 请根据需求替换为 UserAgent/GlobalAgent/GlobalDaemon
	// srv, err := daemon.New(name, description, SystemDaemon, dependencies...)

	// 替换为 MyDaemon
	srv, err := daemon.New(name, description, MyDaemon, dependencies...)
...
```

最后是编译的时候添加上对应的系统标签：

```bash
# Linux
GOOS=linux go build -o myservice-linux main.go

# Windows
GOOS=windows go build -o myservice-windows.exe main.go

# macOSIntel)
GOOS=darwin go build -o myservice-macos-amd64 main.go

# FreeBSD
GOOS=freebsd go build -o myservice-freebsd main.go
```

## 参考资料

[takama/daemon 开源仓库](https://github.com/takama/daemon)
