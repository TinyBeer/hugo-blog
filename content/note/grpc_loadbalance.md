---
date: "2026-03-23T10:30:15+08:00"
title: "gRPC -- 负载均衡"
tags: ["gRPC", "golang", "loadbalance"]
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

负载均衡是 `gRPC` 的一个重要特性，它允许客户端根据一定的策略将请求分发到多个服务端，平衡各个服务端之间的负载。结合名称解析等功能还能拓展服务端的动态适所容能力。

<!--more-->

`gRPC` 的负载均衡以客户端侧实现为核型，聚焦于每一次调用而非连接（每一次请求发生时执行负载均衡策略）。

## 内置策略

`gRPC` 官方提供两种内置策略供选择：

1. `pick_first`  
   它是 `gRPC` 的默认负载均衡策略。作用是按地址顺序尝试连接，只使用第一个成功连接的实例，后续请求均发往该实例。它并无真正负载均衡，仅做故障转移；适用于 开发 / 测试阶段部署 和 单实例部署。
2. `round_robin`  
   轮询策略：为所有可用地址建立子通道，按顺序轮流分发请求，流量均匀。简单高效，充分利用多实例；需配合服务发现动态更新地址列表。

## 负载均衡策略配置

通过在创建客户端时候添加 `WithDefaultServiceConfig` 选项，我们可以方便的切换负载均衡策略。  
比如配置之使用内置轮询策略：

```golang
...
cc, err := grpc.NewClient(target,
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
    grpc.WithTransportCredentials(insecure.NewCredentials()))
...
```

## 补充

除了使用内置负载均衡策略外，`gRPC` 还可以进行以下方式实现负载均衡：

1. `xDS`  
   原理：通过 `xDS` 协议 从控制平面（如 `Istio`、`Envoy`）动态获取 `后端地址`、`负载均衡策略`、`超时、重试、熔断规则`。  
   特点：集中式流量治理、策略统一下发、支持动态扩缩容；是 `gRPC` 云原生负载均衡的事实标准。  
   适用：K8s、服务网格、大规模微服务。I  

2. 代理/服务端负载均衡  
   原理：客户端连接代理服务器，在代理服务器上进行负载均衡。  
   优势：客户端无感知、集中管控、便于限流/监控。  
   劣势：增加一跳、长连接管理复杂。  

## 参考资料

[gRPC官方文档](https://grpc.io/docs/)  
[gRPC-go 仓库](https://github.com/grpc/grpc-go)
