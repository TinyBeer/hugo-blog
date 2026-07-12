---
date: '2026-07-05T19:25:48+08:00'
title: 'slog 使用教程'
tags: ['Golang', 'slog']
categories: "笔记"
description: "Go 标准库 log/slog 结构化日志包使用教程，涵盖基本用法、Handler、自定义 Handler、性能优化及最佳实践。"
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

## 概述

Go 1.21 正式引入 `log/slog` 标准库，为 Go 生态带来了官方原生的**结构化日志**支持。在此之前，社区方案林立（logrus、zap、zerolog 等），各自定义了一套接口和字段格式，项目之间难以互通。

slog 的核心理念：

- **结构化** —— 日志以键值对形式记录，便于机器解析和检索
- **可扩展** —— 通过 `slog.Handler` 接口，可以接入任意后端
- **高性能** —— 内置零分配设计，关键路径避免内存分配
- **标准库** —— 零依赖，未来 Go 项目可以把它作为日志门面

> **💡 重点:** slog 定位于**日志门面（logging facade）**，而非全功能日志框架。它定义了一套统一的接口和基本实现，复杂场景（日志轮转、异步写入等）可以通过实现自定义 Handler 或结合第三方库完成。

## 快速开始

以下代码演示 slog 最基本的用法：

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // 默认输出到 os.Stderr，格式为 TextHandler
    slog.Info("服务启动", "addr", ":8080", "env", "production")

    // 创建 JSONHandler，输出到 stdout
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    logger.Info("用户登录", "user_id", 12345, "ip", "192.168.1.1")
}
```

输出：

```
time=2026-07-05T19:25:48.000+08:00 level=INFO msg="服务启动" addr=:8080 env=production
{"time":"2026-07-05T19:25:48.000+08:00","level":"INFO","msg":"用户登录","user_id":12345,"ip":"192.168.1.1"}
```

## 日志级别

slog 内置四个级别，均为 `slog.Level` 类型（底层是 `int`）：

| 级别 | 值 | 说明 |
|---|---|---|
| `LevelDebug` | -4 | 调试信息 |
| `LevelInfo` | 0 | 常规信息 |
| `LevelWarn` | 4 | 警告 |
| `LevelError` | 8 | 错误 |

每个级别对应一个快捷方法：`slog.Debug`、`slog.Info`、`slog.Warn`、`slog.Error`。

```go
slog.Debug("查询结果", "count", 42)          // 默认 Handler 级别为 Info，这条不会输出
slog.Info("请求开始", "path", "/api/users")
slog.Warn("磁盘使用率超过 80%", "used", 83)
slog.Error("数据库连接失败", "err", err)
```

### 自定义级别

可以通过在 `slog.Level` 值上加减偏移量来定义更细粒度的级别：

```go
const (
    LevelTrace = slog.LevelDebug - 4   // -8
    LevelFatal = slog.LevelError + 4   // 12
)

// 自定义级别的日志需要用 LogAttrs 或 Log 方法输出
slog.LogAttrs(context.Background(), LevelTrace, "trace 信息")
slog.LogAttrs(context.Background(), LevelFatal, "fatal 信息")
```

> **⚠️ 注意:** 自定义级别没有对应的快捷方法，必须使用 `Log` / `LogAttrs` 调用。另外，默认 Handler 不识别自定义级别的名称，会显示为数值（如 `-8`）。要显示自定义名称，需要在 Handler 的 `ReplaceAttr` 选项中处理。

## 结构化日志

slog 的核心数据结构是 `slog.Record`，它由一条消息和若干 `slog.Attr`（属性）组成。

### 键值对模式

最常用的方式是直接在日志方法后追加键值对：

```go
slog.Info("订单创建",
    "order_id", "ORD-20260705-001",
    "amount", 99.9,
    "currency", "CNY",
    "paid", true,
)
```

### Attr 构造方法

`slog.Attr` 是一对 `(key, Value)`。可以通过一组构造方法创建类型安全的 Attr：

```go
slog.Info("商品信息",
    slog.String("name", "无线鼠标"),
    slog.Int("price", 12900),         // 单位：分
    slog.Float64("weight", 65.5),
    slog.Bool("in_stock", true),
    slog.Time("created_at", time.Now()),
    slog.Duration("duration", 3*time.Hour),
    slog.Uint64("sku_id", 9876543210),
    slog.Any("metadata", map[string]string{"color": "black"}),
)
```

**为什么使用类型化 Attr？** 类型化的 `Attr`（如 `slog.Int`）会保留值的原始类型信息，便于 JSON Handler 输出正确的 JSON 类型（数字而不是字符串），也方便下游分析系统根据类型做索引。

### 日志分组

使用 `slog.Group` 将相关字段组织在一起：

```go
slog.Info("用户注册",
    slog.String("email", "user@example.com"),
    slog.Group("profile",
        slog.String("nickname", "小明"),
        slog.Int("age", 28),
        slog.String("city", "北京"),
    ),
)
```

JSON 输出中，分组字段会嵌套为一个子对象：

```json
{
  "time": "...",
  "level": "INFO",
  "msg": "用户注册",
  "email": "user@example.com",
  "profile": {
    "nickname": "小明",
    "age": 28,
    "city": "北京"
  }
}
```

> **💡 重点:** 分组不仅在视觉上组织日志字段，还方便下游日志系统（如 Elasticsearch、Loki）进行嵌套查询。建议将同一业务实体的字段放在一个 Group 下。

### LogAttrs 与 Log

`Log` 和 `LogAttrs` 是底层方法，当字段是动态构造时更高效：

```go
// LogAttrs — 参数为 []Attr，没有 interface{} 装箱
slog.LogAttrs(context.Background(), slog.LevelInfo, "批量导入完成",
    slog.Int("total", 1000),
    slog.Int("success", 998),
    slog.Int("failed", 2),
)

// Log — 参数为 alternating key-value，与快捷方法相同
slog.Log(context.Background(), slog.LevelWarn, "限流触发",
    "threshold", 1000,
    "current", 1500,
)
```

> **⚠️ 注意:** `LogAttrs` 在性能敏感路径上优于 `Log` 和快捷方法，因为它避免了 `any` / `interface{}` 装箱。在高频日志场景（如每秒数千条请求日志）推荐使用。

## Logger

### 创建 Logger

通过 `slog.New` 传入一个 Handler 来创建 Logger：

```go
jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
textLogger := slog.New(slog.NewTextHandler(os.Stderr, nil))
```

### 默认 Logger

slog 包级别的函数（`slog.Info`、`slog.With` 等）操作的是**默认 Logger**：

```go
// 查看当前默认 Logger
slog.Info("hello")  // 使用默认 Logger

// 替换默认 Logger
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))
// 此后所有 slog.Info 等调用都使用 JSON 格式输出
```

> **💡 重点:** 建议在 `main.main()` 或 `init()` 中尽早调用 `slog.SetDefault` 替换默认 Logger，确保整个应用的日志格式统一。库代码应避免调用 `slog.SetDefault`，以免影响引入方。

### Logger 方法链

Logger 提供三个派生方法，返回新的 Logger，不会修改原 Logger：

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

// With — 为 Logger 附加固定字段
l1 := logger.With("service", "user-svc", "version", "v1.2.0")
l1.Info("服务健康检查通过")  // 每条日志自动携带 service、version 字段

// WithGroup — 为 Logger 附加分组
l2 := logger.WithGroup("request")
l2.Info("收到请求", "method", "GET", "path", "/api")

// WithAttrs — 与 With 相同，但接收 []Attr，性能更优
l3 := logger.WithAttrs([]slog.Attr{
    slog.String("trace_id", "abc-123"),
    slog.String("span_id", "def-456"),
})
l3.Info("trace 注入完成")
```

> **⚠️ 注意:** `With` 和 `WithGroup` 返回的 Logger 会**增加每条日志记录的处理开销**，因为每次调用 `Handle` 时都要合并派生字段。不要在循环内反复调用 `With` 创建新的 Logger，建议在初始化阶段一次性创建。

## Handler

### 内置 Handler

slog 内置了两个 Handler：

- **`slog.TextHandler`** —— 人类可读的 key=value 格式，适合开发环境
- **`slog.JSONHandler`** —— JSON 格式，适合生产环境和日志收集系统

```go
// TextHandler（默认）
h1 := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
slog.New(h1).Info("text 格式")

// JSONHandler
h2 := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelWarn})
slog.New(h2).Warn("json 格式")
```

### Handler 选项

通过 `slog.HandlerOptions` 控制 Handler 的行为：

```go
opts := &slog.HandlerOptions{
    // 设置最低输出级别，低于此级别的日志被丢弃
    Level: slog.LevelInfo,

    // 是否在日志中输出源码位置（file:line）
    AddSource: true,

    // 自定义属性处理函数，可以修改/删除/替换 Attr
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
        // 删除 sensitive 字段
        if a.Key == "password" || a.Key == "token" {
            return slog.Attr{}
        }
        // 将时间格式化为更易读的形式
        if a.Key == "time" {
            return slog.String("time", a.Value.Time().Format("2006-01-02 15:04:05"))
        }
        return a
    },
}

logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
logger.Info("用户登录", "email", "user@example.com", "password", "secret123")
```

### 最小级别过滤

通过 `Level` 选项可以设置全局最小级别，也可以通过 `slog.LevelVar` 实现**运行时动态调整**：

```go
var level = new(slog.LevelVar) // 初始值为 LevelInfo

func main() {
    level.Set(slog.LevelDebug) // 运行时切换为 Debug 级别

    h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: level,
    })
    slog.SetDefault(slog.New(h))

    slog.Debug("这条 debug 现在可以输出了")
}
```

> **💡 重点:** 使用 `slog.LevelVar`（指针类型）可以实现运行时动态调整日志级别，无需重启进程。配合 HTTP Handler 暴露 `/debug/level` 端点，可以在生产环境按需调高或降低日志级别。

## 自定义 Handler

实现 `slog.Handler` 接口即可创建自定义 Handler：

```go
type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}
```

以下是一个带颜色的控制台 Handler 实现：

```go
type ColorHandler struct {
    out   io.Writer
    level slog.Leveler
}

func (h *ColorHandler) Enabled(_ context.Context, l slog.Level) bool {
    return l >= h.level.Level()
}

func (h *ColorHandler) Handle(_ context.Context, r slog.Record) error {
    color := map[slog.Level]string{
        slog.LevelDebug: "\033[36m", // 青色
        slog.LevelInfo:  "\033[32m", // 绿色
        slog.LevelWarn:  "\033[33m", // 黄色
        slog.LevelError: "\033[31m", // 红色
    }
    reset := "\033[0m"

    levelStr := r.Level.String()
    c, ok := color[r.Level]
    if !ok {
        c = "\033[0m"
    }

    fmt.Fprintf(h.out, "%s[%s]%s %s", c, levelStr, reset, r.Message)

    r.Attrs(func(a slog.Attr) bool {
        fmt.Fprintf(h.out, " %s=%v", a.Key, a.Value.Any())
        return true
    })
    fmt.Fprintln(h.out)
    return nil
}

func (h *ColorHandler) WithAttrs(attrs []slog.Attr) slog.Handler { return h }
func (h *ColorHandler) WithGroup(name string) slog.Handler       { return h }

// 使用
slog.SetDefault(slog.New(&ColorHandler{out: os.Stdout, level: slog.LevelDebug}))
slog.Info("彩色日志输出")
slog.Error("错误信息是红色的")
```

> **⚠️ 注意:** 自定义 Handler 必须正确处理 `WithAttrs` 和 `WithGroup`。如果只是简单包装（如上面示例），在调用 `Logger.With()` 后再调用 `Handler.Handle()` 时，附加的属性和分组需要通过 `Record.AddAttrs()` 等方法传递给 `Handle`。更完整的实现建议参考 `slog.JSONHandler` 源码。

### 多路输出

结合 `io.MultiWriter` 可以将日志同时输出到多个目的地：

```go
func main() {
    logFile, _ := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
    multi := io.MultiWriter(os.Stdout, logFile)

    logger := slog.New(slog.NewJSONHandler(multi, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)

    slog.Info("这条日志同时写入终端和文件")
}
```

## Context 与日志

slog 原生支持 `context.Context` 传递 Logger，这在请求链路中非常有用：

```go
// 将 Logger 存入 context
ctx := slog.WithContext(context.Background(),
    slog.With("trace_id", "trace-abc", "request_id", "req-456"),
)

// 在函数深层取出 Logger
func handleRequest(ctx context.Context) {
    logger := slog.FromContext(ctx)
    logger.Info("处理请求", "handler", "GetUser")
}
```

> **💡 重点:** 使用 `slog.WithContext` / `slog.FromContext` 替代手动传递 Logger 参数，可以让中间件（如 HTTP 中间件）透明地注入追踪信息，业务代码无需关心日志参数传递。

更常见的使用方式是搭配 HTTP 中间件：

```go
func LogMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := slog.WithContext(r.Context(),
            slog.With(
                "method", r.Method,
                "path", r.URL.Path,
                "remote", r.RemoteAddr,
            ),
        )
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(w http.ResponseWriter, r *http.Request) {
    logger := slog.FromContext(r.Context())
    logger.Info("开始处理请求")
    // ...
}
```

## 日志记录器集成

### 桥接标准库 log

`slog` 包提供了 `NewLogLogger` 将 slog 适配到标准库 `log.Logger`：

```go
// 创建一个使用 slog JSONHandler 的标准库 Logger
slogLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
stdLogger := slog.NewLogLogger(slogLogger.Handler(), slog.LevelInfo)

// 第三方库（如 net/http、database/sql）内部使用这个 Logger
stdLogger.Println("来自标准库的日志")
```

### 桥接 zap / zerolog

slog 生态提供了双向适配：

```go
import "github.com/samber/slog-zap"

func main() {
    zapLogger, _ := zap.NewProduction()
    // 将 slog Handler 路由到 zap 后端
    slog.SetDefault(slog.New(slogzap.Option{Level: slog.LevelDebug, Logger: zapLogger}.NewZapHandler()))
    slog.Info("这条日志由 zap 后端处理")
}
```

相反方向，也可以通过第三方库将 zap/zerolog 的日志路由到 slog Handler。

## 第三方生态

基于 slog 的优秀第三方库：

| 库 | 说明 |
|---|---|
| [samber/slog-xxx](https://github.com/samber/slog) | 全家桶：slog-zap、slog-sentry、slog-datadog、slog-otel、slog-multi 等 |
| [lmittmann/tint](https://github.com/lmittmann/tint) | 带颜色的终端输出 Handler |
| [phsym/console-slog](https://github.com/phsym/console-slog) | 类似 logrus 风格的彩色控制台 Handler |
| [veqryn/slog-context](https://github.com/veqryn/slog-context) | 通过 context 增强日志（请求 ID、追踪等） |
| [go-slog/otelslog](https://github.com/go-slog/otelslog) | OpenTelemetry 集成 Handler |

## 性能

### 设计哲学：零分配与懒求值

slog 在设计上做了大量性能优化：

1. **懒求值** —— 如果日志因级别过滤被丢弃，参数不会被处理
2. **`slog.Attr` 避免装箱** —— 类型化 Attr（`slog.Int`、`slog.String` 等）直接携带原始类型，不经过 `interface{}`
3. **Record 复用** —— Handler 的 `Handle` 方法收到的是值类型 `Record`，鼓励 Handler 使用 `Record.Attrs` 方法遍历属性而非复制

```go
// 推荐：类型化 Attr，避免装箱
slog.Debug("查询完成",
    slog.Int("count", n),  // 类型化，零装箱
    slog.Duration("latency", d),
)

// 不推荐：any 装箱
slog.Debug("查询完成",
    "count", n,      // 隐式转为 any
    "latency", d,
)
```

> **⚠️ 注意:** 懒求值意味着如果参数是**函数调用**（如 `slog.Any("data", expensiveFunc())`），该函数**总是**会被调用，即使日志被过滤。避免将昂贵的函数调用作为日志参数。

### 性能对比

> 以下数据基于 Go 1.22 基准测试，仅供参考。实际性能依日志量、Handler 选择和环境而异。

| 库 | 单条日志耗时 | 内存分配 |
|---|---|---|
| slog (TextHandler) | ~200ns | 0 alloc/op |
| slog (JSONHandler) | ~400ns | 0 alloc/op |
| zerolog | ~100ns | 0 alloc/op |
| zap (Sugared) | ~2μs | ~2 allocs/op |
| logrus | ~5μs | ~8 allocs/op |

> **💡 重点:** slog 在性能上接近 zerolog（两者都采用了零分配设计），远优于 logrus。对于大多数应用场景，slog 自身的性能已经足够，无需为了性能替换为 zap/zerolog。选择日志库时应优先考虑**标准库的统一性和可扩展性**。

## 最佳实践

### 区分 Info 与 Debug

生产环境建议将默认级别设为 Info，Debug 仅在开发或故障排查时开启。

```go
// main.go
var logLevel = new(slog.LevelVar)

func init() {
    logLevel.Set(slog.LevelInfo)
    slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: logLevel,
    })))
}

// 通过环境变量或配置文件动态调整
func SetLogLevel(level string) {
    switch strings.ToLower(level) {
    case "debug":
        logLevel.Set(slog.LevelDebug)
    case "info":
        logLevel.Set(slog.LevelInfo)
    case "warn":
        logLevel.Set(slog.LevelWarn)
    case "error":
        logLevel.Set(slog.LevelError)
    }
}
```

### 使用 Group 组织字段

将同一实体的字段分组，方便下游分析和排查：

```go
slog.Info("支付回调",
    slog.String("order_id", "ORD-20260705-001"),
    slog.Group("payment",
        slog.String("channel", "wechat"),
        slog.Float64("amount", 99.9),
        slog.String("trade_no", "WX20260705001"),
    ),
    slog.Group("user",
        slog.Int64("user_id", 10001),
        slog.String("vip_level", "gold"),
    ),
)
```

### 全局级别动态调整

通过 HTTP 端点暴露日志级别调整能力，方便线上排查：

```go
func main() {
    http.HandleFunc("/debug/level", func(w http.ResponseWriter, r *http.Request) {
        if r.Method == http.MethodGet {
            fmt.Fprintf(w, "current log level: %s\n", logLevel.Level().String())
            return
        }
        level := r.URL.Query().Get("level")
        SetLogLevel(level)
        fmt.Fprintf(w, "log level changed to: %s\n", level)
    })
    log.Fatal(http.ListenAndServe(":6060", nil))
}
```

### 日志轮转

slog 自身不处理日志轮转，推荐结合 `lumberjack` 实现：

```go
import "gopkg.in/natefinch/lumberjack.v2"

func main() {
    w := &lumberjack.Logger{
        Filename:   "logs/app.log",
        MaxSize:    100,  // 每个文件最大 100MB
        MaxBackups: 10,   // 保留 10 个旧文件
        MaxAge:     30,   // 保留 30 天
        Compress:   true, // 压缩旧文件
    }

    logger := slog.New(slog.NewJSONHandler(w, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
    slog.SetDefault(logger)
}
```

## 参考资料

- [Official slog package](https://pkg.go.dev/log/slog)
- [Go Blog: Structured Logging with slog](https://go.dev/blog/slog)
- [slog 使用详解](https://blog.axiaoxin.com/post/go-slog-usage)
- [slog 源码分析](https://www.notdamn.work/posts/slog_source_code_analysis/)
