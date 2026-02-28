---
date: "2026-02-28T14:54:00+08:00"
title: "gRPC -- 现代化高性能RPC框架"
tags: ["gRPC", "rpc"]
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

<!-- hello world -->

## RPC 模式

### 简单 RPC

### 服务器流式 RPC

### 客户端流式 RPC

### 双向流式 RPC

## 参考资料

[gRPC官方文档](https://grpc.io/docs/)
