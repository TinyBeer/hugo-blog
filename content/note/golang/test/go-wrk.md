---
date: "2026-01-20T10:11:08+08:00"
title: "go-wrk -- HTTP服务压力测试工具"
tags: ["wrk", "Benchmark", "Golang"]
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

`go-wrk` 是一款基于 `Go` 语言开发的 HTTP 基准测试工具，灵感源自 C 语言编写的 [`wrk`](https://github.com/wg/wrk)。

<!--more-->

它通过 `goroutine` 和调度器实现异步 `IO` 与并发，以简洁代码提供高性能的 `HTTP` 服务压测能力，适用于 `API`、`Web` 服务的吞吐量、延迟等指标评估。

## 安装

```bash
go install github.com/tsliwowicz/go-wrk@latest
```

## 参数

```bash
go-wrk -v
Version: 0.10
```

| 参数     | 说明                                         | 默认值  |
| :------- | :------------------------------------------- | :------ |
| `-H`     | 添加请求头，可以使用多个 `-H` 添加多个请求头 |         |
| `-M`     | HTTP 请求方法                                | `GET`   |
| `-T`     | 请求超时时间 单位 毫秒                       | `1000`  |
| `-body`  | 请求体数据，可以填字符串或者 `@文件名`       |         |
| `-c`     | 并发连接数量（协程数）                       | `10`    |
| `-ca`    | `ca` 文件                                    |         |
| `-cert`  | `ca certification` 文件                      |         |
| `-d`     | 测试持续时间，单位 秒                        | `10`    |
| `-http`  | 使用 `http/2`                                | `true`  |
| `-key`   | 私钥文件                                     |         |
| `-no-c`  | 不启用 `Compression`                         | `false` |
| `-no-ka` | 不启用 `KeepAlive`                           | `false` |
| `-no-vr` | 跳过证书验证                                 | `false` |
| `-redir` | 允许重定向                                   | `false` |
| `-v`     | 打印版本信息                                 | `false` |

## 使用示例

使用 `Gin` 框架搭建一个 `API` 服务器进行演示：

```golang
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()

	router.GET("/test", func(c *gin.Context) {
		c.String(http.StatusOK, "ok")
	})

	router.Run(":9999") // 监听 9999 端口
}

```

开启4个协程，测试持续 5s：

```bash
go-wrk -d 5 -c 4 http://localhost:9999/test
Running 5s test @ http://localhost:9999/test
  4 goroutine(s) running concurrently
220066 requests in 4.865927008s, 21.20MB read
Requests/sec:		45225.91
Transfer/sec:		4.36MB
Overall Requests/sec:	43979.96
Overall Transfer/sec:	4.24MB
Fastest Request:	36µs
Avg Req Time:		87µs
Slowest Request:	31.492ms
Number of Errors:	0
10%:			45µs
50%:			49µs
75%:			50µs
99%:			51µs
99.9%:			51µs
99.9999%:		51µs
99.99999%:		51µs
stddev:			162µs
```

## 参考资料

[go-wrk 代码仓库](https://github.com/tsliwowicz/go-wrk)
