---
date: "2026-03-06T15:47:39+08:00"
title: "gRPC -- 错误处理"
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

`gRPC` 中的错误信息是基于 状态码（`code`）、消息（`message`）、错误详情 (`details`) 的结构化设计，它能够精准传递跨语言、跨平台的错误信息。

<!--more-->

## 错误模型

在 `go` 语言中，`gRPC` 错误信息的结构如下：

```golang
// The `Status` type defines a logical error model that is suitable for
// different programming environments, including REST APIs and RPC APIs. It is
// used by [gRPC](https://github.com/grpc). Each `Status` message contains
// three pieces of data: error code, error message, and error details.
//
// You can find out more about this error model and how to work with it in the
// [API Design Guide](https://cloud.google.com/apis/design/errors).
type Status struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields

	// The status code, which should be an enum value of
	// [google.rpc.Code][google.rpc.Code].
	Code int32 `protobuf:"varint,1,opt,name=code,proto3" json:"code,omitempty"`
	// A developer-facing error message, which should be in English. Any
	// user-facing error message should be localized and sent in the
	// [google.rpc.Status.details][google.rpc.Status.details] field, or localized
	// by the client.
	Message string `protobuf:"bytes,2,opt,name=message,proto3" json:"message,omitempty"`
	// A list of messages that carry the error details.  There is a common set of
	// message types for APIs to use.
	Details []*anypb.Any `protobuf:"bytes,3,rep,name=details,proto3" json:"details,omitempty"`
}
```

不难看出它是通过 `protobuf` 的 `message` 定义出的：

```proto
message Status {
  // The status code, which should be an enum value of
  // [google.rpc.Code][google.rpc.Code].
  int32 code = 1;

  // A developer-facing error message, which should be in English. Any
  // user-facing error message should be localized and sent in the
  // [google.rpc.Status.details][google.rpc.Status.details] field, or localized
  // by the client.
  string message = 2;

  // A list of messages that carry the error details.  There is a common set of
  // message types for APIs to use.
  repeated google.protobuf.Any details = 3;
}
```

### 状态码（`code`）

`gRPC` 中定义了 17 个状态码（编号 0-16）：

```golang
type Code uint32

const (
  // 成功
  OK Code = 0

  // 请求被取消
  Canceled Code = 1

  // 未知错误
  Unknown Code = 2

  // 参数不合法
  InvalidArgument Code = 3

  // 请求超时
  DeadlineExceeded Code = 4

  // 资源不存在
  NotFound Code = 5

  // 已存在相同的资源
  AlreadyExists Code = 6

  // 权限不足被拒绝访问
  PermissionDenied Code = 7

  // 资源枯竭，剩下的容量不足以使用，如磁盘容量不足
  ResourceExhausted Code = 8

  // 不满足执行条件
  FailedPrecondition Code = 9

  // 请求被中断
  Aborted Code = 10

  // 操作访问超出限制范围
  OutOfRange Code = 11

  // 调用没有实现
  Unimplemented Code = 12

  // 系统内部错误
  Internal Code = 13

  // 服务不可用
  Unavailable Code = 14

  // 数据丢失
  DataLoss Code = 15

  // 没有通过认证
  Unauthenticated Code = 16

  _maxCode = 17
)
```

### 消息

错误消息（`message`）一段简短的描述性文字，用来简略说明异常原因。

### 详细信息

详细信息（`Details`）是一组使用 `protobuf` 定义的 `message`，用来携带更加详尽的错误信息。尽管 `Status` 中的状态码和消息已经可以覆盖大部分场景了，但是总有一些情况是无法完全满足需求的。这时我们就可以使用 `Details` 来对错误细节进行补充。

## 快速使用

### 构造错误信息

在 `go` 语言中 `gRPC` 提供了以下方法在服务端快速构造错误信息：

```golang
// func Err(c codes.Code, msg string) error
success := status.Error(codes.OK, "request success")

// func Errorf(c codes.Code, format string, a ...interface{}) error
notFound := status.Errorf(codes.InvalidArgument, "invalid argument name: %s", name)
```

如果我们在服务端实现远程调用时未使用 `Status` 结构返回 `error`，那么 `gRPC` 会自动为我们封装一个状态码为 `Unknown` 的错误信息（`Status`）。

### 解析错误信息

在客户端我们可以使用 `google.golang.org/grpc/status` 包中的以下方法解析错误信息：

```golang
// 将 error 转换为 Status
func FromError(err error) (s *Status, ok bool)

func Convert(err error) *Status

// 从err中获取状态码
func Code(err error) codes.Code
```

### 错误详情

错误详情的操作有以下几种方式：

```golang
// 添加错误详情
func (s *Status) WithDetails(details ...proto.Message) (*Status, error)

// 获取错误信息中的错误详情列表
// s := status.Convert(err)
// for _, d := range s.Details() {
// 	 ...
// }

func (s *Status) Details() []any

```

## 参考资料

[gRPC官方文档](https://grpc.io/docs/)  
[gRPC-go 仓库](https://github.com/grpc/grpc-go)
