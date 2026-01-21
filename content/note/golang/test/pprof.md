---
date: "2026-01-20T22:28:54+08:00"
title: "Golang -- 运行时数据分析 pprof"
tags: ["Golang", "pprof"]
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

`pprof` 是 `Go` 语言自带的一个运行时数据工具，它提供了对程序不同方面性能数据的采集和分析功能，主要涉及 `CPU` 性能、内存分配、阻塞情况、锁竞争等方面。

<!--more-->

通过分析这些数据，开发者可以了解程序在运行过程中各个函数的资源消耗情况，进而有针对性地进行优化。

## 数据采集

### cpu

`cpu` 性能数据的采集在所有数据的采集中是相对特殊的，它需要对 `cpu` 数据进行多次采样，故而 `runtime/pprof` 包中提供了一套独立的方法来进行处理。一下使用一个求斐波那契数进行演示：

```golang
package main

import (
	"fmt"
	"os"
	"runtime/pprof"
)

func Fibo(n int) int {
	if n <= 2 {
		return 1
	}
	return Fibo(n-1) + Fibo(n-2)
}

func main() {
	// 创建一个文件用于保存 CPU 分析数据
	f, err := os.Create("cpu.prof")
	if err != nil {
		panic(err)
	}
	defer f.Close()
	// 开始 CPU 分析
	err = pprof.StartCPUProfile(f)
	if err != nil {
		panic(err)
	}
	defer pprof.StopCPUProfile()
	// 斐波那契数列计算执行

	fmt.Println(Fibo(43))
}
```

运行代码后，我们可以获得一个 `cup` 采样结果文件 `cpu.prof`。这个文件如何使用将在后续数据分析部分介绍。

### go test

`go test` 命令有两个参数和 `pprof` 相关，它们分别指定生成的 `CPU` 和 `Memory profiling` 保存的文件：

- `-cpuprofile`：`cpu`数据要保存的文件地址
- `-memprofile`：内存数据要报文的文件地址

通过添加这两个参数可以在基准测试时进行采样：

```
go test -bench . -cpuprofile=cpu.prof
go test -bench . -memprofile=mem.prof
```

### 其他

对一其他运行时数据，`runtime/pprof` 包中提供了 `Lookup` 函数进行数据采集。`Lookup` 支持以下参数：

| 参数           | 说明                                                                   |
| :------------- | :--------------------------------------------------------------------- |
| `goroutine`    | 记录程序里所有正在运行 / 等待的 `goroutine` 调用栈                     |
| `heap`         | 程序当前正在使用的内存分配情况（抽样统计）                             |
| `allocs`       | 程序运行至今所有内存分配记录（抽样统计）                               |
| `threadcreate` | 记录哪些代码逻辑触发了新系统线程的创建                                 |
| `block`        | 记录 `goroutine` 因锁 / 通道等同步操作被卡住的调用栈                   |
| `mutex`        | 记录哪些 `goroutine` 占用了有竞争的互斥锁（导致其他 `goroutine` 等待） |

数据采样代码参考：

```golang
package main

import (
	"fmt"
	"log"
	"os"
	"runtime/pprof"
)

func Fibo(n int) int {
	if n <= 2 {
		return 1
	}
	return Fibo(n-1) + Fibo(n-2)
}

func main() {
	fmt.Println(Fibo(43))

	// 这里根据需求选择需要采集的数据类型
	w, _ := os.Create("heap.prof")
	heapProfile := pprof.Lookup("heap")
	err := heapProfile.WriteTo(w, 0)
	if err != nil {
		log.Fatal(err)
	}
}

```

其中 `WriteTo` 方法的第二个参数可选则以下三种：

- `0`：写入压缩后的 `Protobuf` 数据，没有可读性
- `1`：写入文本格式的数据，能够阅读，`http` 接口返回的就是这一种数据
- `2`：仅 `goroutine` 可用，表示打印 `panic` 风格的堆栈信息

## 数据分析

`Go` 提供了 `go tool pprof` 来帮助我们分析采集到的数据。用法如下：

```bash
# format 指定输出格式
# source 代表数据源 可以是文件 也可以是 api 地址
go tool pprof <format> [options] [binary] <source> ...

# 忽略 format 会开启一个命令行交互界面
go tool pprof [options] [binary] <source> ...

# 忽略 format 并且添加 -http 参数 会开启一个 web 交互界面
go tool pprof -http [host]:[port] [options] [binary] <source> ...
```

### 命令行交互

这里使用之前采集的 `cpu.prof` 演示使用，其他资源数据用法类似。

命令行交互界面中常用的命令：

- `top` 输出资源占用最高的条目，可以添加参数显示使用最高的 n 条
- `list` 通过正则表达式匹配条目，显示资源使用信息
- `pdf/svg/png/gif` 生成图片展示的资源使用报告，需要安装 [Graphvia](https://graphviz.org/download/)
- `web` 进入 `web` 交互界面
- `q` 退出交互界面，同作用的命令还有 `quit`, `exit`

> 如果没有 `Graphvia`，可以使用 `tool pprof -dot cpu.prof > cpu.dot` 生成 `dot` 文件然后在线可视化工具（如：[GraphvizOnline](https://dreampuf.github.io/GraphvizOnline)）查看。

使用方法如下：

```bash
go tool pprof cpu.prof
File: pprof
Build ID: 81b7131d57eea75676f9756f82c39e068e07783e
Type: cpu
Time: 2026-01-20 22:46:47 CST
Duration: 1.40s, Total samples = 1.23s (87.69%)
Entering interactive mode (type "help" for commands, "o" for options)
...
(pprof) top
Showing nodes accounting for 1.23s, 100% of 1.23s total
      flat  flat%   sum%        cum   cum%
     1.23s   100%   100%      1.23s   100%  main.Fibo
         0     0%   100%      1.23s   100%  main.main
         0     0%   100%      1.23s   100%  runtime.main
...

top 1
Showing nodes accounting for 1.23s, 100% of 1.23s total
Showing top 1 nodes out of 3
      flat  flat%   sum%        cum   cum%
     1.23s   100%   100%      1.23s   100%  main.Fibo
...

(pprof) list Fibo
Total: 1.23s
ROUTINE ======================== main.Fibo in /home/beer/workspace/test/pprof/main.go
     1.23s      2.04s (flat, cum) 165.85% of Total
     600ms      600ms      9:
     100ms      100ms     10:func Fibo(n int) int {
     110ms      110ms     11:	if n <= 2 {
         .          .     12:		return 1
     420ms      1.23s     13:	}
         .          .     14:	return Fibo(n-1) + Fibo(n-2)
         .          .     15:}
         .          .     16:
         .          .     17:func main() {
         .          .     18:	fmt.Println(Fibo(43))
...
```

### web页面交互

通过 `-http  [host]:[port]` 参数开启 `web` 交互界面，`host` 默认为 `localhost`，这里同样需要 [Graphvia](https://graphviz.org/download/)。

```bash
go tool pprof -http :9999 cpu.prof
Serving web UI on http://localhost:9999
...
```

## 接入采集API

### 注册采集API

`net/http/pprof` 包将性能数据采集功能包装成了 `http` 接口，并注册到了默认路由中：

```golang
package pprof

import ...

func init() {
    http.HandleFunc("/debug/pprof/", Index)
    http.HandleFunc("/debug/pprof/cmdline", Cmdline)
    http.HandleFunc("/debug/pprof/profile", Profile)
    http.HandleFunc("/debug/pprof/symbol", Symbol)
    http.HandleFunc("/debug/pprof/trace", Trace)
}
...
```

对于使用 `http.DefaultServeMux` 路由直接引入包即可：

```golang
package main

import (
  "net/http"
    // 记得要导入这个包
  _ "net/http/pprof"
)

func main() {
    go func(){
        http.ListenAndServe(":9999", nil)
    }
    for {
        Do()
    }
}
```

对于自定义路由需要手动进行注册：

```golang
...
r.HandleFunc("/debug/pprof/", pprof.Index)
r.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
r.HandleFunc("/debug/pprof/profile", pprof.Profile)
r.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
r.HandleFunc("/debug/pprof/trace", pprof.Trace)
...
```

对于 `Gin` 框架，社区提供了 `github.com/gin-contrib/pprof` 库供使用：

```golang
...
pprof.Register(router)
...
```

### 使用采集API

手动采样：

对于 `net/http/pprof`、`github.com/gin-contrib/pprof` 包接入的采集API,可以进入 `http://127.0.0.1:<port>/debug/pprof` （`port` 是服务监听端口）网页根据网页提示操作。

自动采样：
我们可以将 `go tool pprof <format> [options] [binary] <source> ...` 中的 `source` 替换为对应的 `api` 地址进行自动采样分析：

```bash
go tool pprof -http :8888 http://127.0.0.1:9999/debug/pprof/heap
```

使用 `API` 采样时可以通过参数更细致的控制采样，详细说明可以运行 `go tool pprof` 查看。

## 参考资料

[Go官方文档 -- pprof包](https://pkg.go.dev/runtime/pprof#WriteHeapProfile)  
[Golang 中文学习文档 -- 性能分析](https://golang.halfiisland.com/essential/senior/140.pprof.html)  
[Go性能调优](https://www.liwenzhou.com/posts/Go/pprof/)
