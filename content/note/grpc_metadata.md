---
date: "2026-03-04T15:05:04+08:00"
title: "gRPC -- 元数据 metadata"
tags: ["gRPC", "golang"]
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

`gRPC` 元数据（`metadata`）是一个额外的信息通道，用于在服务器和客户端端传递 `rpc` 相关信息（通常是非业务核心的），通常包括 鉴权信息、追踪信息、自定义信息等。

<!--more-->

`gRPC` 中元数据是一组键值对，使用 `HTTP/2` 的 `header` 实现。其中键是 `string` 类型，由字母、数字和特殊字符 `-`、`_`、`.` 组成，大小写不敏感，但不能使用 `grpc-` 为前缀（这代表 `gRPC` 的保留字段），值是 `[]string` 类型或者二进制类型。元数据可在 `请求/响应` 的 `开始/结束` 阶段传递。

`gRPC` 中元数据可以分为 `header` 和 `trailer` 两种。`header` 可以在 请求/响应 前传递。 `trailer` 只能在服务端完成响应后发送。

`gRPC` 元数据可以在多种场景下应用，如：

1. 鉴权
   元数据可以用来向服务端传递鉴权信息，这可以在多种鉴权机制中使用。如：`OAuth2` 或者 `JWT` 可以使用标准的 `HTTP Authorization 头`。
2. 追踪
   元数据可以用来携带一些追踪信息（如 链路ID），用来帮助追踪一个请求的完整执行流程。
3. 自定义数据传递
   元数据可以用来在客户端和服务端间传递一些自定义头信息，这可以用来实现 负载均衡、请求限流、传递详细的错误信息等。
4. 业务数据传递
   虽然并不推荐这样做，但是元数据确实可以用来传递一些业务数据。

[!WARNING] 服务器可以限制请求头（元数据）的大小，通常推荐大小不超过 `8 KiB`。

## Go 语言中使用元数据

在 `go` 语言中 `gRPC` 元数据的处理需要使用 `google.golang.org/grpc/metadata` 库，其中元数据的类型定义如下：

```golang
type MD map[string][]string
```

我们通过对 键 添加 `-bin` 告知框架 值 类型为二进制数据，`gRPC` 框架会自动对这些数据进行 `base64` 编码处理，包括 在传输是进行编码 和 在接收时进行解码。

### 创建元数据

在 `Go` 语言中我们通常使用一下两种方法创建 `gRPC` 元数据。

```golang
md := metadata.New(map[string]string{"key1": "val1", "key2": "val2"})
```

```golang
md := metadata.Pairs(
    "key", "string value",
    "key-bin", string([]byte{96, 102}), // this binary data will be encoded (base64) before sending
                                        // and will be decoded after being transferred.
)
```

### 客户端发送元数据

在客户端发送元数据有两种方法，推荐使用 `metadata.AppendToOutgoingContext` 向 `context.Context` 中追加元数据（元数据不存在时创建元数据，存在时则合并元数据）：

```golang
// create a new context with some metadata
ctx := metadata.AppendToOutgoingContext(ctx, "k1", "v1", "k1", "v2", "k2", "v3")

// later, add some more metadata to the context (e.g. in an interceptor)
ctx := metadata.AppendToOutgoingContext(ctx, "k3", "v4")

// make unary RPC
response, err := client.SomeRPC(ctx, someRequest)

// or make streaming RPC
stream, err := client.SomeStreamingRPC(ctx)
```

另一个替代方案是使用 `metadata.NewOutgoingContext` 创建新的元数据，它会清理掉原有元数据，速度也更慢一些：

> [!IMPORTANT] `FromOutgoingContext` 仅在客户端使用，不能在服务端使用。

```golang
// create a new context with some metadata
md := metadata.Pairs("k1", "v1", "k1", "v2", "k2", "v3")
ctx := metadata.NewOutgoingContext(context.Background(), md)

// later, add some more metadata to the context (e.g. in an interceptor)
send, _ := metadata.FromOutgoingContext(ctx)
newMD := metadata.Pairs("k3", "v3")
ctx = metadata.NewOutgoingContext(ctx, metadata.Join(send, newMD))

// make unary RPC
response, err := client.SomeRPC(ctx, someRequest)

// or make streaming RPC
stream, err := client.SomeStreamingRPC(ctx)
```

### 客户端读取元数据

客户端可以读取到 `header` 和 `trailer` 两种元数据。
对于普通响应，我们通过在请求中添加 `option` 读取元数据：

```golang
var header, trailer metadata.MD // variable to store header and trailer
r, err := client.SomeRPC(
    ctx,
    someRequest,
    grpc.Header(&header),    // will retrieve header
    grpc.Trailer(&trailer),  // will retrieve trailer
)

// do something with header and trailer
```

对于流式响应，我们使用 `stream` 对象的 `Header` 和 `Trailer` 方法获取元数据：

```golang
stream, err := client.SomeStreamingRPC(ctx)

// retrieve header
header, err := stream.Header()

// retrieve trailer
trailer := stream.Trailer()
```

### 服务端读取元数据

对于服务端读取元数据，我们需要使用到 `metadata.FromIncomingContext` 从 `context.Context` 中进行解析：

普通请求：

```golang
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    md, ok := metadata.FromIncomingContext(ctx)
    // do something with metadata
}
```

流式请求：

```golang
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    md, ok := metadata.FromIncomingContext(stream.Context()) // get context from stream
    // do something with metadata
}
```

### 服务端发送元数据

对于普通响应，我们通过 `grpc.SetHeader` 和 `grpc.SetTrailer` 添加元数据：

```golang
func (s *server) SomeRPC(ctx context.Context, in *pb.someRequest) (*pb.someResponse, error) {
    // create and set header
    header := metadata.Pairs("header-key", "val")
    grpc.SetHeader(ctx, header)
    // create and set trailer
    trailer := metadata.Pairs("trailer-key", "val")
    grpc.SetTrailer(ctx, trailer)
}
```

对于流式响应，我使用 `stream` 对象的 `SetHeader` 和 `SetTrailer` 方法添加元数据：

```golang
func (s *server) SomeStreamingRPC(stream pb.Service_SomeStreamingRPCServer) error {
    // create and set header
    header := metadata.Pairs("header-key", "val")
    stream.SetHeader(header)
    // create and set trailer
    trailer := metadata.Pairs("trailer-key", "val")
    stream.SetTrailer(trailer)
}
```

<!-- TODO 服务端口使用拦截器处理元数据 -->

## 参考资料

[gRPC官方文档](https://grpc.io/docs/)  
[gRPC-go 仓库](https://github.com/grpc/grpc-go)
