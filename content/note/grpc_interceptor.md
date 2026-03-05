---
date: "2026-03-05T13:58:27+08:00"
title: "gRPC -- 拦截器（中间件）"
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

拦截器是 `gRPC` 中用于实现大量 `RPC` 方法公共行为的机制。它也常被称为过滤器（`filter`） 或者 中间件（`middleware`）。

<!--more-->

拦截器常被用于以下场景中：

- 元数据处理
- 日志输出
- 错误注入（一种测试手段）
- 数据缓存
- 指标采集
- 认证鉴权
- 。。。

> `gRPC-go` 社区中已经整理了很多常用的拦截器（中间件）供大家直接取用 [传送门](https://github.com/grpc-ecosystem/go-grpc-middleware)

## 拦截器分类

拦截器按照使用位置可以分为 客户端拦截器 和 服务端拦截器，按照处理 RPC 类型可以分为 普通（Unary）拦截器 和 流式拦截器。

|         | 客户端           | 服务端           |
| :------ | :--------------- | :--------------- |
| 普通RPC | 普通客户端拦截器 | 普通服务端拦截器 |
| 流式RPC | 流式客户端拦截器 | 流式服务端拦截器 |

## 普通客户端拦截器

普通客户端拦截器函数签名如下：

```golang
func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
```

客户端普通拦截器的实现通常会被分解为三个部分：处理前、`RPC` 方法执行、处理后。

对于处理前，拦截器可以通过输入参数了解到本次方法调用的信息，包括 上下文、方法名、请求信息以及配置的请求选项（Options）。利用这些信息我们可以对本次方法调用进行一些预处理。
对于方法执行，我们使用 `grpc.UnaryInvoker` 执行操作。
对于处理后，我们通常进行一些错误处理、日志输出、返回信息处理等。

以下是一个官方示例：

```golang
...
const fallbackToken = "some-secret-token"
...
// unaryInterceptor is an example unary interceptor.
func unaryInterceptor(ctx context.Context, method string, req, reply any, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
	var credsConfigured bool
	for _, o := range opts {
		_, ok := o.(grpc.PerRPCCredsCallOption)
		if ok {
			credsConfigured = true
			break
		}
	}
	if !credsConfigured {
		opts = append(opts, grpc.PerRPCCredentials(oauth.TokenSource{
			TokenSource: oauth2.StaticTokenSource(&oauth2.Token{AccessToken: fallbackToken}),
		}))
	}
	start := time.Now()
	err := invoker(ctx, method, req, reply, cc, opts...)
	end := time.Now()
	logger("RPC: %s, start time: %s, end time: %s, err: %v", method, start.Format("Basic"), end.Format(time.RFC3339), err)
	return err
}
```

在这个示例中，拦截器检查了本地调用是否配置了鉴权信息，如果没有，则使用默认鉴权信息（以 `fallbackToken` 为 `AccessToken` 的 `auth2` 鉴权信息）。在方法执行结束后，拦截器打印了执行时间以及错误信息相关的日志。

## 流式客户端拦截器

流式客户端拦截器函数签名如下：

```golang
func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
```

实现一个流式客户端拦截器通常包括两部分：处理前 和 流操作。

处理前部分和普通客户端拦截器中相似，这里就不过多赘述了。对于流操作的部分，我们通常使用一个潜入 `ClientStream` 的结构体对原始客户端流对象进行封装。以下是一个官方示例：

```golang
// wrappedStream  wraps around the embedded grpc.ClientStream, and intercepts the RecvMsg and
// SendMsg method call.
type wrappedStream struct {
	grpc.ClientStream
}

func (w *wrappedStream) RecvMsg(m any) error {
	logger("Receive a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ClientStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m any) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ClientStream.SendMsg(m)
}

func newWrappedStream(s grpc.ClientStream) grpc.ClientStream {
	return &wrappedStream{s}
}

// streamInterceptor is an example stream interceptor.
func streamInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
	var credsConfigured bool
	for _, o := range opts {
		_, ok := o.(*grpc.PerRPCCredsCallOption)
		if ok {
			credsConfigured = true
			break
		}
	}
	if !credsConfigured {
		opts = append(opts, grpc.PerRPCCredentials(oauth.TokenSource{
			TokenSource: oauth2.StaticTokenSource(&oauth2.Token{AccessToken: fallbackToken}),
		}))
	}
	s, err := streamer(ctx, desc, cc, method, opts...)
	if err != nil {
		return nil, err
	}
	return newWrappedStream(s), nil
}
```

`wrappedStream` 通过实现（重载） `RecvMsg`、`SendMsg` 方法的方式，对消息的接收和处理进行拦截，实现打印消息信息和时间信息的效果。

## 普通服务端拦截器

普通服务器拦截器和普通客户端拦截器类似，仅在入参信息上有所区别，函数签名如下：

```golang
func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

以下是一个官方示例：

```golang
// valid validates the authorization.
func valid(authorization []string) bool {
	if len(authorization) < 1 {
		return false
	}
	token := strings.TrimPrefix(authorization[0], "Bearer ")
	// Perform the token validation here. For the sake of this example, the code
	// here forgoes any of the usual OAuth2 token validation and instead checks
	// for a token matching an arbitrary string.
	return token == "some-secret-token"
}

func unaryInterceptor(ctx context.Context, req any, _ *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
	// authentication (token verification)
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, errMissingMetadata
	}
	if !valid(md["authorization"]) {
		return nil, errInvalidToken
	}
	m, err := handler(ctx, req)
	if err != nil {
		logger("RPC failed with error: %v", err)
	}
	return m, err
}
```

在这个示例中，拦截器检查了元数据中的鉴权信息。

## 流式服务端拦截器

流式服务器拦截器和流式客户端拦截器同样类似，仅在流处理部分有所区别，函数签名如下：

```golang
func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```

具体使用方法可以参考以下示例：

```golang
// wrappedStream wraps around the embedded grpc.ServerStream, and intercepts the RecvMsg and
// SendMsg method call.
type wrappedStream struct {
	grpc.ServerStream
}

func (w *wrappedStream) RecvMsg(m any) error {
	logger("Receive a message (Type: %T) at %s", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m any) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.SendMsg(m)
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
	return &wrappedStream{s}
}

func streamInterceptor(srv any, ss grpc.ServerStream, _ *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	// authentication (token verification)
	md, ok := metadata.FromIncomingContext(ss.Context())
	if !ok {
		return errMissingMetadata
	}
	if !valid(md["authorization"]) {
		return errInvalidToken
	}

	err := handler(srv, newWrappedStream(ss))
	if err != nil {
		logger("RPC failed with error: %v", err)
	}
	return err
}
```

## 拦截器安装

我们使用 `grpc.WithUnaryInterceptor` 和 `grpc.WithStreamInterceptor` 选项（`Options`）在创建客户端对象时安装拦截器。

```golang
...
// Set up a connection to the server.
conn, err := grpc.NewClient(*addr,
grpc.WithTransportCredentials(creds),
grpc.WithUnaryInterceptor(unaryInterceptor),
grpc.WithStreamInterceptor(streamInterceptor))
...
```

使用 `grpc.UnaryInterceptor` 和 `grpc.StreamInterceptor` 选项（`Options`） 在创建服务端对象时安装拦截器。

```golang
...
s := grpc.NewServer(
    grpc.Creds(creds),
    grpc.UnaryInterceptor(unaryInterceptor),
    grpc.StreamInterceptor(streamInterceptor))
...
```

## 参考资料

[gRPC官方文档](https://grpc.io/docs/)  
[gRPC-go 仓库](https://github.com/grpc/grpc-go)
