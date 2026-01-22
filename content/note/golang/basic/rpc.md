---
date: "2026-01-22T10:37:38+08:00"
title: "Golang -- net/rpc 包"
tags: ["Golang", "rpc"]
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

`rpc` 包是 `Go` 语言官方提供的一种将对象方法通过网络或者其他 `I/O` 连接对外暴露（对外提供服务）的方法。

<!--more-->

`rpc` 包默认通过 `encoding/gob` 进行序列化/反序列化，当然也可以手动选择其他序列化方式。

> `encoding/gob` 是 `Go` 语言特有的二进制序列化/反序列化包，专门用于在 `Go` 程序之间传输数据（也可用于本地持久化）。它相比 `JSON `XML`这类文本格式，具有更高的性能和更小的体积，且能原生支持`Go`的特有类型（如结构体、切片、map、接口等），是`Go` 程序间通信的 “专属序列化工具”。

## HTTP 服务

`rpc` 通过在服务（`http`, `tcp` 等）上登记对象，将一个对象转换成一个对外可见的服务，服务名称为该对象的类型名。对象登记成服务后，对象上满足条件的方法就可以进行远程调用了。一个服务可以登记多个对象，但同一个类型的对象只能登记一次，否则会报错。

对外暴露的方法需要一下条件：

1. 登记的对象类型需要是可导出的（首字母大写）
2. 方法是可导出的
3. 方法需要有两个参数，参数类型是可导出类型或者内建类型
4. 方法的第二个参数是指针类型
5. 方法的返回一个 `error`

方法签名： `func (t *T) MethodName(argType T1, replyType *T2) error`

下面使用一个 `Arith` 对象（对外提供 `Multiply` 和 `Divide` 两个方法）作为示例，示例工程结构如下：

```bash
 tree
.
├── client
│   └── main.go
├── common
│   └── args.go
├── go.mod
└── service
    ├── arith.go
    └── main.go
```

其中 `service`, `client` 分别存放服务端和客户端代码，`common` 用于存放通行双放需要用到的公共代码（参数定义）。

### 公共部分

`args.go` 代码如下：

```golang
package common

// 入参
type Args struct {
	A int
	B int
}

// 触发返回
type Quotient struct {
	Quo int
    Rem int
}
```

### 服务端

`arith.go` 定义需要对外提供调用的方法：

```golang
package main

import (
	"errors"
	"net_rpc/common"
)

type Arith int

func (t *Arith) Multiply(args *common.Args, reply *int) error {
	*reply = args.A * args.B
	return nil
}

func (t *Arith) Divide(args *common.Args, quo *common.Quotient) error {
	if args.B == 0 {
		return errors.New("不能除零")
	}
	quo.Quo = args.A / args.B
	quo.Rem = args.A % args.B
	return nil
}
```

`service/main.go` 进行服务注册并对外提供调用服务：

`rpc` 通过 `Register` 登记服务。本示例中使用 `http` 默认路由对外提供服务，所以使用 `rpc.HandleHTTP()` 注册路由。

```golang
package main

import (
	"log"
	"net"
	"net/http"
	"net/rpc"
)

func main() {
	arith := new(Arith)
    // 注册 rpc 服务
	rpc.Register(arith)
    // 向 http 包中的默认路由注册服务
	rpc.HandleHTTP()
	l, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("listen error:", err)
	}
    // 启动服务
	err = http.Serve(l, nil)
	if err != nil {
		log.Fatal("Serve error:", err)
	}
}
```

### 客户端

客户端通过 `DialHTTP` 连接服务端，进行远程调用。这里 `rpc` 包提供了两种调用方式，一种是同步调用（`client.Call`），一种是一步调用 （`client.Go`），它们的第一个参数指定远程调用的方法，格式为： `类型名称.方法名`。第二个参数则是方法的第一个参数。具体代码如下：

```golang
package main

import (
	"fmt"
	"log"
	"net/rpc"
	"net_rpc/common"
)

func main() {
	client, err := rpc.DialHTTP("tcp", "127.0.0.1:1234")
	if err != nil {
		log.Fatal("dialing:", err)
	}

	// 同步调用
	args := &common.Args{
		A: 7,
		B: 8,
	}
	var reply int
	err = client.Call("Arith.Multiply", args, &reply)
	if err != nil {
		log.Fatal("arith error:", err)
	}
	fmt.Printf("Arith: %d*%d=%d\n", args.A, args.B, reply)

	// 异步调用
	quotient := new(common.Quotient)
	divCall := client.Go("Arith.Divide", args, quotient, nil)
    // 等待异步处理结束
	replyCall := <-divCall.Done
	// 检查异步 处理 error
	if replyCall.Error != nil {
		log.Fatal("arith error:", replyCall.Error)
	}
	fmt.Printf("Arith: %d/%d=%d %d%%%d=%d\n", args.A, args.B, quotient.Quo, args.A, args.B, quotient.Rem)
}
```

### 测试

服务端：

```bash
go run ./service
```

客户端：

```bash
go run ./client
Arith: 7*8=56
Arith: 7/8=0 7%8=7
```

## TCP 协议

将之前的示例代码改为 `TCP` 协议，我们只需要修改监听创建和连接创建即可，其他代码无需改动即可工作。

### 服务端

对于服务端只需要创建一个 `tcp` 协议的 `net.Listener` ，然后将其交给 `rpc.Accept` 处理即可。

```golang
package main

import (
	"log"
	"net"
	"net/rpc"
)

func main() {
	arith := new(Arith)
	rpc.Register(arith)
	rpc.Register(arith) // 注册RPC服务
	l, e := net.Listen("tcp", ":1234")
	if e != nil {
		log.Fatal("listen error:", e)
	}
	rpc.Accept(l)
}
```

### 客户端

对于客户端，使用 `rpc.Dial` 替换 `rpc.DialHTTP` 连接服务端：

```golang
package main

import (
	"fmt"
	"log"
	"net/rpc"
	"net_rpc/common"
)

func main() {
	client, err := rpc.Dial("tcp", "127.0.0.1:1234")
	if err != nil {
		log.Fatal("dial error:", err)
	}

	// 同步调用
	args := &common.Args{
		A: 7,
		B: 8,
	}
	var reply int
	err = client.Call("Arith.Multiply", args, &reply)
	if err != nil {
		log.Fatal("arith error:", err)
	}
	fmt.Printf("Arith: %d*%d=%d\n", args.A, args.B, reply)

	// 异步调用
	quotient := new(common.Quotient)
	divCall := client.Go("Arith.Divide", args, quotient, nil)
	replyCall := <-divCall.Done // 等待调用返回
	// 错误处理
	if replyCall.Error != nil {
		log.Fatal("arith error:", replyCall.Error)
	}
	fmt.Printf("Arith: %d/%d=%d %d%%%d=%d\n", args.A, args.B, quotient.Quo, args.A, args.B, quotient.Rem)
}

```

## JSON 序列化

官方还提供了使用 `json` 序列化消息的方法，由于使用比较少，这里就不进行演示了。想要了解的可以参考 [net/rpc/jsonrpc 包](https://pkg.go.dev/net/rpc/jsonrpc)。

## 参考资料

[Go官方文档 -- net/rpc 包](https://pkg.go.dev/net/rpc)
