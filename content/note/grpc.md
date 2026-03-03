---
date: "2026-02-28T14:54:00+08:00"
title: "gRPC -- 现代化高性能RPC框架"
tags: ["gRPC", "rpc"]
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

`gRPC` 是由 `Google` 开发的一款高性能、跨语言的远程过程调用（`RPC`）框架，基于 `HTTP/2` 协议设计，使用 `Protocol Buffers（Protobuf）` 作为接口定义语言（`IDL`）和数据序列化格式。

<!--more-->

## 核心特性

- **`跨语言 / 跨平台`**：支持几乎所有主流编程语言（`Go`、`Java`、`Python`、`C++`、`JavaScript` 等），不同语言编写的服务可以无缝通信。
- **`高性能`**：基于 `HTTP/2` 实现，支持多路复用（一个连接处理多个请求）、二进制帧传输、头部压缩，性能远优于传统的 REST API。
- **`强类型接口`**：通过 Protobuf 定义服务和数据结构，编译时就能检查类型错误，避免运行时的类型问题。
- **`多种 RPC 模式`**：满足不同业务场景需求：
  - 简单 RPC：客户端发请求，服务端返回响应（一对一）。
  - 服务器流式 RPC：客户端发一次请求，服务端返回流式响应（比如实时日志推送）。
  - 客户端流式 RPC：客户端流式发请求，服务端返回一次响应（比如批量数据上传）。
  - 双向流式 RPC：客户端和服务端都可以流式收发数据（比如实时聊天、视频通话）。

## 核心工作流程

1. 定义接口：用 `Protobuf` 编写 `.proto` 文件，声明服务（方法名、参数、返回值）和数据结构。
2. 生成代码：通过 `protoc` 编译器 + 对应语言的 `gRPC` 插件，生成服务端 / 客户端的代码骨架。
3. 实现服务端：编写业务逻辑，实现 `Protobuf` 定义的服务方法。
4. 调用客户端：通过生成的客户端代码，像调用本地函数一样调用远程服务。

## 快速开始

### 依赖安装

首先，我们需要准备好编译工具 `protoc` 。

> [protoc 下载地址](https://github.com/protocolbuffers/protobuf/releases)
> 解压完成后需要将 `bin` 目录添加到环境变量中，以便系统能识别到 `protoc` 工具。

安装 `go` 语言编译插件：

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

这里也需要将 `GOPATH` 添加都环境变量中，否则无法识别到对应插件。

### 项目结构

在开始编码前，我们先定好项目结构：

```bash
.
├── client
│   └── main.go
├── hello_world
│   └── hello_world.proto
└── server
    └── main.go
```

我们在更目录中使用 `go mod init grpc_example` 初始化项目。

### 编写 proto 文件

接下来我们根据上方的核心工作流逐步完成代码编写，这里我们先进行 `hello_world.proto` 文件的编写：

```proto
syntax = "proto3";

package hello_world;

option  go_package = "grpc_example/hello_world";

// 定义 HelloWorld 服务
service HelloWorld {
  // 定义 HelloWorld 接口
  rpc HelloWorld(HelloWorldReq) returns (HelloWorldResp){};
}

// HelloWorld 接口请求消息
message HelloWorldReq {
  string name = 1;
}

// HelloWorld 接口响应消息
message HelloWorldResp {
  string reply = 1;
}
```

这里定义了一个 `HelloWorld` 服务，包含一个 `HelloWorld` 接口，请求消息中包含一个字符串类型的 `name` 字段，响应消息中包含一个字符串类型的 `reply` 字段。

### 编译 proto 文件

接下来使用 `protoc` 编译 `hello_world.proto` 生成 `go` 代码。

```bash
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative hello_world/hello_world.proto
```

执行完成后，会在 `hello_world` 文件夹下生成两个文件 `hello_world.pb.go` 和 `hello_world_grpc.pb.go`。

### 服务端代码实现

在 `server/main.go` 添加如下代码：

```golang
package main

import (
	"context"
	"flag"
	"fmt"
	"grpc_example/hello_world"
	"log"
	"net"

	"google.golang.org/grpc"
)

const (
	addr = ":9999"
)

type HelloWorldServer struct {
	// 添加所有接口未实现处理
	hello_world.UnimplementedHelloWorldServer
}

// 实现 HelloWorld 接口
func (s *HelloWorldServer) HelloWorld(_ context.Context, req *hello_world.HelloWorldReq) (*hello_world.HelloWorldResp, error) {
	name := req.GetName()
	if name == "" {
		name = "world"
	}

	return &hello_world.HelloWorldResp{
		Reply: fmt.Sprintf("Hello %s!", name),
	}, nil
}

func main() {
	flag.Parse()
	// 创建 tcp 监听
	lis, err := net.Listen("tcp", addr)
	if err != nil {
		log.Fatal(err)
	}

  // 实例化一个 grpc server
	svc := grpc.NewServer()
  // 注册 HelloWorld 服务
	hello_world.RegisterHelloWorldServer(svc, new(HelloWorldServer))

	// run rpc server
	log.Printf("server listen on %s\n", addr)
	err = svc.Serve(lis)
	if err != nil {
		log.Fatal(err)
	}
}
```

### 客户端代码实现

在 `client/main.go` 添加如下代码：

```golang
package main

import (
	"context"
	"flag"
	"grpc_example/hello_world"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

const (
	addr = ":9999"
)

var (
	name = flag.String("name", "", "client target name")
)

func main() {
	flag.Parse()
	// 建立连接  无加密
	conn, err := grpc.NewClient(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	// 创建客户端
	client := hello_world.NewHelloWorldClient(conn)
	// 执行调用
	resp, err := client.HelloWorld(context.Background(), &hello_world.HelloWorldReq{
		Name: *name,
	})
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("receive rpc response: %+v", resp)

}

```

## 测试

启动服务端：

```bash
go run ./server/main.go
2026/03/03 12:12:02 server listen on :9999
```

执行客户端：

```bash
go run ./client/main.go
2026/03/03 12:12:50 receive rpc response: reply:"Hello world!"

go run ./client/main.go -name tom
2026/03/03 12:13:01 receive rpc response: reply:"Hello tom!"
```

## 流式RPC

`gRPC` 允许定义四种类型的服务接口，包括 `简单RPC`、`服务端流式RPC`、`客户端流式RPC` 以及 `双向流式RPC`。上文中快速开始使用的就是一个 `简单RPC` 接口。

> 为了避免大量的代码冗余，后文的介绍中仅针对不同 流式RPC 差异化部分进行编码演示。
> 后文中的演示代码使用的是泛型接口，如果不希望使用泛型接口可以参考 [文档](https://grpc.io/docs/languages/go/generated-code-old/)

### 服务端流式RPC

`服务端流式RPC` 是在用户端发送一条请求，而服务端可能返回多条响应时候使用的接口。在 `proto` 文件中，使用 `stream` 关键字说明服务端的返回是流式的。

- 接口定义：

  ```proto
  // say hello to one group
  rpc HelloGroup(HelloReq) returns (stream HelloResp){};
  ```

- 请求处理代码：

  ```golang
  func (s *GreeterServer) HelloGroup(req *greeter.HelloReq, stream grpc.ServerStreamingServer[greeter.HelloResp]) error {
    group := req.GetName()
    var names []string
    switch group {
    case "animal":
      names = []string{"cat", "dog", "mouse"}
    case "fruit":
      names = []string{"apple", "banana", "orange"}
    default:
      names = []string{}
    }

    for _, name := range names {
      err := stream.Send(&greeter.HelloResp{
        Reply: fmt.Sprintf("Hello %s!", name),
      })
      if err != nil {
        return err
      }
    }
    return nil
  }
  ```

- 客户端请求代码：

  ```golang
  ...
  respStream, err := client.HelloGroup(context.Background(), &greeter.HelloReq{
  	Name: "animal",
  })
  if err != nil {
  	log.Fatal(err)
  }

  for {
  	resp, err := respStream.Recv()
  	if err == io.EOF { // 判断服务端是否结束发送
  		break
  	}
  	if err != nil {
  		log.Fatalf("client.HelloGroup failed: %v", err)
  	}
  	log.Printf("receive rpc response: %+v", resp.String())
  }
  ...
  ```

### 客户端流式RPC

`客户端流式RPC` 是一种客户端发送多条请求消息，而服务端在客户端完成发送后返回一条总结性的响应的接口。

- 接口定义：

  ```proto
  // say hello to many persons
  rpc HelloMulti(stream HelloReq) returns (HelloResp){};
  ```

- 请求处理代码：

  ```golang
  func (s *GreeterServer) HelloMulti(stream grpc.ClientStreamingServer[greeter.HelloReq, greeter.HelloResp]) error {
    startTime := time.Now()
    var names []string
    for {
      req, err := stream.Recv()
      if err == io.EOF { // 按端客户端是否结束发送
        log.Printf("receve names[%s] from %v to %v\n", names, startTime, time.Now())
        return stream.SendAndClose(&greeter.HelloResp{ // 发送总结下响应，并关闭请求
          Reply: fmt.Sprintf("Hello %s!", strings.Join(names, ",")),
        })
      }
      if err != nil {
        return err
      }
      name := req.GetName()
      log.Printf("receive hello multi name[%s]\n", name)
      names = append(names, name)
    }
  }
  ```

- 客户端请求代码：

  ```golang
  ...
  reqStream, err := client.HelloMulti(context.Background())
  if err != nil {
  	log.Fatal(err)
  }
  for _, name := range []string{"tom", "jerry", "jj"} {
  	err = reqStream.Send(&greeter.HelloReq{
  		Name: name,
  	})
  	if err != nil {
  		log.Fatal(err)
  	}
  }
  resp2, err := reqStream.CloseAndRecv() // 结束请求，并接受总结性响应
  if err != nil {
  	log.Fatal(err)
  }

  log.Printf("receive rpc response: %+v", resp2.String())
  ...
  ```

### 双向流式RPC

`双向流式RPC` 是一种客户端和服务端消息就能为流式的接口。

- 接口定义：

  ```proto
  // say many hello
  rpc ManyHello(stream HelloReq) returns (stream HelloResp){};
  ```

- 请求处理代码：

  ```golang
  func (s *GreeterServer) ManyHello(stream grpc.BidiStreamingServer[greeter.HelloReq, greeter.HelloResp]) error {
    greeted := make(map[string]struct{})
    for {
      in, err := stream.Recv()
      if err == io.EOF { // 判断客户端是否结束请求
        // 关闭请求
        return nil
      }
      if err != nil {
        return err
      }

      name := in.GetName()
      if _, has := greeted[name]; has {
        err = stream.Send(&greeter.HelloResp{
          Reply: fmt.Sprintf("%s has greeted!", name),
        })
      } else {
        greeted[name] = struct{}{}
        err = stream.Send(&greeter.HelloResp{
          Reply: fmt.Sprintf("Hello %s!", name),
        })
      }
      if err != nil {
        return err
      }
    }
  }
  ```

- 客户端请求代码：

  ```golang
  ...
  ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
  defer cancel()
  biStream, err := client.ManyHello(ctx)
  if err != nil {
    log.Fatalf("client.ManyHello failed: %v", err)
  }
  waitc := make(chan struct{}) // 创建管道避免程序过早结束
  go func() { // 启动协程处理服务端响应
    for {
      in, err := biStream.Recv()
      if err == io.EOF { // 判断服务端是否发送完成
        // read done.
        // 结束响应消息处理的协程
        close(waitc)
        return
      }
      if err != nil {
        log.Fatalf("client.ManyHello failed: %v", err)
      }
      log.Printf("receive rpc response: %+v", in.String())
    }
  }()
  // 发送多条请求
  for _, name := range []string{"tom", "jerry", "kitty", "tom"} {
    if err := biStream.Send(&greeter.HelloReq{
      Name: name,
    }); err != nil {
      log.Fatalf("client.ManyHello: greet name(%s) failed: %v", name, err)
    }
  }
  biStream.CloseSend() // 告知服务端请求发送完毕
  <-waitc // 利用chan特性 避免程序过早结束
  ...
  ```

## 参考资料

[gRPC官方文档](https://grpc.io/docs/)
