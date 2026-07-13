---
date: "2026-07-13T14:38:54+08:00"
title: "Golang -- pprof 性能分析工具使用指南"
tags: ["Golang", "pprof", "性能分析"]
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

`pprof` 是 Go 语言自带的性能分析工具，提供对程序不同方面性能数据的采集和分析功能<!--more-->，主要涉及 **CPU 性能**、**内存分配**、**阻塞情况**、**锁竞争** 等方面。通过分析这些数据，开发者可以了解程序在运行过程中各个函数的资源消耗情况，进而有针对性地进行优化。

## 工作流程

pprof 的使用分为三个阶段：

1. **采样**：程序运行期间，按一定时间间隔对程序状态进行采样，记录函数调用栈、内存分配等信息。
2. **数据生成**：将采样数据保存到 `.prof` 文件中，包含程序运行时的详细性能信息。
3. **数据展示**：使用 `go tool pprof` 读取数据文件，以文本或可视化方式展示分析结果。

## 命令速查

| 命令            | 说明                            | 示例                    |
| --------------- | ------------------------------- | ----------------------- |
| `top`           | 显示资源占用最多的函数          | `top`、`top10`          |
| `list <func>`   | 显示函数源码并标注每行耗时      | `list main.fibonacci`   |
| `web`           | 可视化调用图（需安装 graphviz） | `web`                   |
| `dot`           | 导出 dot 格式，可粘贴到在线工具 | `dot > cpu.dot`         |
| `peek <func>`   | 显示函数的调用者和被调用者      | `peek main.fibonacci`   |
| `disasm <func>` | 反汇编函数并标注热点            | `disasm main.fibonacci` |
| `quit`          | 退出 pprof 交互模式             | `quit`                  |

## CPU 性能分析

利用 `pprof` 可以方便地得到一段代码中各个阶段占用的 CPU 时间，通常选取占用最多的函数进行针对性分析，达到优化性能的目的。下面以斐波那契数列计算为例进行演示。

### 原始代码

```go
package main

import (
    "fmt"
    "time"
)

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    for i := 0; i < 40; i++ {
        go fibonacci(i)
    }
    fmt.Println("Calculating Fibonacci numbers...")
    time.Sleep(5 * time.Second)
}
```

### 添加 CPU 性能分析

引入 `runtime/pprof` 包，在程序中插入采样逻辑：

```go
package main

import (
    "os"
    "runtime/pprof"
    // ... 其他导入
)

func main() {
    // 创建文件用于保存 CPU 分析数据
    f, err := os.Create("cpu.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    // 开始 CPU 分析
    if err := pprof.StartCPUProfile(f); err != nil {
        panic(err)
    }
    defer pprof.StopCPUProfile()

    // ... 业务代码
}
```

运行程序生成性能文件：

```bash
go run main.go
```

### 使用 pprof 分析

```bash
go tool pprof cpu.prof
```

进入交互界面后常用命令：

**`top`** — 显示 CPU 占用最多的函数：

```bash
(pprof) top
Showing nodes accounting for 920ms, 100% of 920ms total
    flat  flat%   sum%        cum   cum%
    920ms   100%   100%      920ms   100%  main.fibonacci
```

**`list`** — 显示函数源码并标注每行 CPU 使用情况：

```bash
(pprof) list main.fibonacci
Total: 920ms
ROUTINE ======================== main.fibonacci in /home/beer/workspace/test/pprof/main.go
    920ms      1.34s (flat, cum) 145.65% of Total
    240ms      240ms     10:func fibonacci(n int) int {
    130ms      130ms     11:   if n <= 1 {
    50ms       50ms     12:           return n
        .          .     13:   }
    500ms      920ms     14:   return fibonacci(n-1) + fibonacci(n-2)
```

**`web`** — 打开可视化调用图，需安装 `graphviz`。也可以用 `dot > cpu.dot` 导出后粘贴到在线工具如 [GraphvizOnline](https://dreampuf.github.io/GraphvizOnline) 查看。

## 内存性能分析

利用 `pprof` 可以定位内存分配占用的具体代码，找出内存不合理的地方进行优化。

### 添加内存分析代码

```go
package main

import (
    "fmt"
    "os"
    "runtime/pprof"
    "time"
)

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    fb := []int{0, 1}
    for i := 2; i <= n; i++ {
        fb = append(fb, fb[len(fb)-2]+fb[len(fb)-1])
    }
    time.Sleep(time.Second * 6) // 保证采样时内存未被释放
    return fb[n]
}

func main() {
    for i := 0; i < 80; i++ {
        go fibonacci(i)
    }
    fmt.Println("Calculating Fibonacci numbers...")
    time.Sleep(time.Second * 5)

    // 创建文件并写入内存分析数据
    f, err := os.Create("mem.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    if err := pprof.WriteHeapProfile(f); err != nil {
        panic(err)
    }
}
```

### 使用 pprof 分析

```bash
go tool pprof mem.prof
```

```bash
(pprof) top
Showing nodes accounting for 512.03kB, 100% of 512.03kB total
    flat  flat%   sum%        cum   cum%
512.03kB   100%   100%   512.03kB   100%  main.fibonacci
```

使用 `list` 查看具体内存分配行：

```bash
(pprof) list main.fibonacci
  Total: 512.03kB
  ROUTINE ======================== main.fibonacci in /home/beer/workspace/test/pprof/main.go
  512.03kB   512.03kB (flat, cum)   100% of Total
          .          .     10:func fibonacci(n int) int {
          .          .     11:   if n <= 1 {
          .          .     12:           return n
          .          .     13:   }
          .          .     14:   fb := []int{0, 1}
          .          .     15:   for i := 2; i <= n; i++ {
  512.03kB   512.03kB     16:           fb = append(fb, fb[len(fb)-2]+fb[len(fb)-1])
          .          .     17:   }
```

> `top`、`list`、`web`、`dot` 等命令用法与 CPU 分析一致。

## Goroutine 分析

用于排查 goroutine 泄漏。当程序中 goroutine 数量持续增长不释放时，很可能是发生了泄漏。

### 采集方式

通过 Web 接口直接查看（需服务已引入 `net/http/pprof`）：

```bash
# 文本摘要模式（debug=1 显示摘要，debug=2 显示完整堆栈）
curl http://localhost:8888/debug/pprof/goroutine?debug=1

# 下载二进制 profile 文件
curl http://localhost:8888/debug/pprof/goroutine > goroutine.prof
go tool pprof goroutine.prof
```

### 交互分析

```bash
(pprof) top
Showing nodes accounting for 42 goroutines, 100% of 42 total
      flat  flat%   sum%        cum   cum%
        42   100%   100%         0     0%  runtime.gopark
```

```bash
(pprof) list runtime.gopark
```

通过 `list` 查看具体阻塞位置，可以快速定位 goroutine 卡在哪个函数。

### 代码中采集

```go
import "runtime/pprof"

// 获取当前 goroutine 数量
n := runtime.NumGoroutine()

// 将 goroutine 堆栈写入文件
f, _ := os.Create("goroutine.prof")
pprof.Lookup("goroutine").WriteTo(f, 1)
f.Close()
```

## Block / Mutex 分析

用于分析 goroutine 阻塞（channel 操作、锁等待）和锁竞争情况。

### 启用采样

Block 和 Mutex 分析默认关闭，需要在程序中显式启用：

```go
import "runtime"

func main() {
    // 阻塞分析：每发生一次阻塞事件采样一次（设为 1 表示全部采样）
    runtime.SetBlockProfileRate(1)

    // 锁竞争分析：每 N 次锁竞争事件采样一次（设为 1 表示全部采样）
    runtime.SetMutexProfileFraction(1)

    // ... 业务代码
}
```

> `SetMutexProfileFraction(1)` 开销较大，生产环境建议使用更大的值（如 100），降低采样频率。

### 采集与分析

通过 Web 接口采集：

```bash
# 阻塞分析
curl http://localhost:8888/debug/pprof/block > block.prof
go tool pprof block.prof

# 锁竞争分析
curl http://localhost:8888/debug/pprof/mutex > mutex.prof
go tool pprof mutex.prof
```

```bash
(pprof) top
Showing nodes accounting for 1.2s, 100% of 1.2s total
    flat  flat%   sum%        cum   cum%
  800ms 66.67% 66.67%     800ms 66.67%  sync.(*Mutex).Lock
  400ms 33.33%  100%     400ms 33.33%  runtime.chanrecv
```

## 对比模式

用 `-base` 参数对比两份 profile，可以快速看出优化前后的差异。

### 使用方式

```bash
# 优化前
go tool pprof cpu.before.prof

# 优化后
go tool pprof cpu.after.prof

# 对比分析（进入交互模式后）
(pprof) top
```

或者一步到位：

```bash
go tool pprof -base cpu.before.prof cpu.after.prof
```

### 输出解读

```bash
(pprof) top
Showing nodes accounting for 520ms, 100% of 520ms difference
    flat  flat%   sum%        cum   cum%
  520ms 100.00% 100.00%     520ms 100.00%  main.fibonacci
```

- 正值表示优化后**增加**的耗时（说明变慢了）
- 负值表示优化后**减少**的耗时（说明变快了）
- `flat` 列直接反映函数自身的性能差异

## Benchmark 联合使用

将 pprof 与 `go test` 的 Benchmark 结合，可以精准分析测试用例的性能。

### 生成 Profile

```bash
# 同时生成 CPU 和内存 profile
go test -bench=BenchmarkFib -benchmem -cpuprofile=bench_cpu.prof -memprofile=bench_mem.prof -count=1

# 运行多次取平均（-count=5）
go test -bench=BenchmarkFib -benchmem -cpuprofile=bench_cpu.prof -count=5
```

### 分析结果

```bash
go tool pprof bench_cpu.prof
(pprof) top
Showing nodes accounting for 2.3s, 100% of 2.3s total
    flat  flat%   sum%        cum   cum%
  1.8s  78.26% 78.26%     1.8s  78.26%  main.fibonacci
```

### 配合 benchstat 对比

```bash
# 安装工具
go install golang.org/x/perf/cmd/benchstat@latest

# 先保存优化前后的 benchmark 结果
go test -bench=BenchmarkFib -count=10 > old.txt
# ... 修改代码 ...
go test -bench=BenchmarkFib -count=10 > new.txt

# 对比差异
benchstat old.txt new.txt
```

输出示例：

```
name         old time/op  new time/op  delta
Fibonacci-8   1.20s ± 2%   0.80s ± 1%  -33.33%  (p=0.000 n=10+10)
```

## 火焰图

火焰图（Flame Graph）是最直观的性能可视化方式。横轴表示采样占比，纵轴表示调用栈深度，宽度越宽说明耗时越多。

### 生成火焰图

方式一：通过 pprof 交互模式导出 SVG：

```bash
go tool pprof cpu.prof
(pprof) flame > flame.svg
```

方式二：使用 `go tool pprof -http` 直接在浏览器中打开（需安装 graphviz）：

```bash
go tool pprof -http=:8080 cpu.prof
```

浏览器会自动打开一个包含火焰图、调用图、Top 视图的 Web 界面。

### 在线工具

也可以将 SVG 文件上传到在线火焰图工具查看，如 [Speedscope](https://www.speedscope.app)。

## 远程采集

直接从线上或远程服务拉取 profile，无需本地生成文件。

### 前提条件

远程服务需要暴露 pprof 接口（参考 [Web 服务中的性能分析](#web-服务中的性能分析) 章节）。

### 采集命令

```bash
# CPU 采样（默认 30 秒）
go tool pprof http://remote-host:8888/debug/pprof/profile?seconds=30

# 内存采样
go tool pprof http://remote-host:8888/debug/pprof/heap

# Goroutine 采样
go tool pprof http://remote-host:8888/debug/pprof/goroutine

# 阻塞采样
go tool pprof http://remote-host:8888/debug/pprof/block
```

> `seconds` 参数仅对 `profile`（CPU）有效，控制采样时长。其他类型直接采样当前快照。

### 注意事项

- 远程采集会触发实际采样，对线上服务有轻微性能影响，建议在低峰期进行。
- 网络延迟可能导致下载超时，可先用 `curl` 确认接口可达：
  ```bash
  curl -o /dev/null -w "%{http_code}" http://remote-host:8888/debug/pprof/
  ```
- 生产环境建议对 pprof 接口做鉴权限制，避免敏感性能数据泄露。

## Web 服务中的性能分析

对于基于 `net/http` 构建的服务，直接引入 `net/http/pprof` 包即可自动添加调试接口：

```go
package main

import (
    "net/http"
    _ "net/http/pprof"
    "time"
)

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    time.Sleep(time.Second * 2)
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    go func() {
        for i := 0; i < 40; i++ {
            go fibonacci(i)
        }
        time.Sleep(5 * time.Second)
    }()

    // 启动 pprof 服务
    go func() {
        http.ListenAndServe("localhost:8888", nil)
    }()

    select {}
}
```

运行后可通过以下 URL 访问：

| URL                                           | 用途                     |
| --------------------------------------------- | ------------------------ |
| `http://localhost:8888/debug/pprof/`          | 查看所有可用分析数据     |
| `http://localhost:8888/debug/pprof/profile`   | CPU 采样，等待完成后下载 |
| `http://localhost:8888/debug/pprof/heap`      | 内存采样，等待完成后下载 |
| `http://localhost:8888/debug/pprof/goroutine` | Goroutine 堆栈           |
| `http://localhost:8888/debug/pprof/block`     | 阻塞分析                 |
| `http://localhost:8888/debug/pprof/mutex`     | 锁竞争分析               |

使用方式示例：

```bash
go tool pprof http://localhost:8888/debug/pprof/profile
```

> **注意**：接口在访问时触发采样，需等待采样完成后才下载结果数据。

### Gin 框架集成

```go
package main

import (
    "net/http"

    "github.com/gin-contrib/pprof"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    pprof.Register(r)

    r.GET("/ping", func(ctx *gin.Context) {
        ctx.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })

    r.Run(":8888")
}
```

## 实例分析：xorm 同步性能问题

### 问题现象

使用 xorm 同步数据库表结构时，执行时间极长。代码由模板生成，对每个表分别调用同步：

```go
func InitSync(orm *xorm.Engine) error {
    var err error

    err = sync(orm, &AccountAccount{})
    if err != nil {
        return fmt.Errorf("sync &AccountAccount error:%v", err)
    }

    // ... 其他表

    err = sync(orm, &Zone{})
    if err != nil {
        return fmt.Errorf("sync &Zone error:%v", err)
    }

    return nil
}
```

### pprof 分析

使用 pprof 分析后得到 CPU 时间占用图：

![sync_cpu](https://gitee.com/tinybeer/im-bed/raw/master/sync_cpu_20260713145030988.png)

中间一条路径是执行最长的路径。沿此路径查阅 xorm 源码，定位到问题：

```go
// xorm.io/xorm@v1.3.2/engine.go
func (engine *Engine) Sync(beans ...interface{}) error {
    session := engine.NewSession()
    defer session.Close()
    return session.Sync(beans...)
}

// xorm.io/xorm@v1.3.2/session_schema.go
func (session *Session) Sync(beans ...interface{}) error {
    engine := session.engine
    // ...
    tables, err := engine.dialect.GetTables(session.getQueryer(), session.ctx)
    // ...
}
```

`engine.dialect.GetTables` 会读取数据库中**所有表结构**。由于同步代码对每个表单独调用 sync，导致重复拉取全部表结构数据，而实际上只需拉取一次。

### 解决方案

将所有表结构以变长参数方式一次性传入：

```go
err := orm.Sync(
    &AccountAccount{},
    // ... 其他表
    &Zone{},
)
```

修改后同步速度大幅提升。

## 参考资料

[https://pkg.go.dev/runtime/pprof](https://pkg.go.dev/runtime/pprof)  
[Golang中文学习文档 -- 性能分析](https://golang.halfiisland.com/essential/senior/140.pprof.html)  
[Go 语言高性能编程 -- pprof 性能分析](https://geektutu.com/post/hpg-pprof.html)  