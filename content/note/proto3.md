---
date: "2026-02-09T16:37:19+08:00"
title: "Proto3"
tags: ["protobuf"]
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

`Proto3` 是 `Google Protocol Buffers`（Protobuf）的第三个版本，2016 年推出，是一种语言中立、平台中立、可扩展的结构化数据序列化机制，用于序列化结构化数据，常用于通信协议、数据存储等场景，相比 `Proto2` 更简洁、高效、跨语言兼容性更强。

<!--more-->

## 应用场景

由于 `protobug3` 其良好的跨语言兼容性、优秀的性能表现被广泛应用在以下领域：

- `RPC` 通信：如 `gRPC` 默认使用 `Proto3` 定义服务接口和消息结构，高效传输数据。
- 数据存储：序列化数据存储到磁盘或数据库，如日志存储、配置文件等。
- 跨语言数据交换：在不同语言编写的系统间传输数据，如 `Java` 后端与 `Go` 客户端通信。
- 移动开发：生成的代码体积小、效率高，适配移动端性能要求。

## 消息

## 枚举类型

## 服务

## Oneof

## 引用

## Any

## 参考资料

[protobuf 参考文档](https://protobuf.dev/programming-guides/proto3/)
