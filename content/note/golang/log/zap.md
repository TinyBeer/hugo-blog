---
date: '2026-07-12T12:15:16+08:00'
title: 'Zap -- 高性能结构化日志库'
tags: ['Golang', 'log']
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


`zap` 是 Uber 团队开发的 Go 语言高性能结构化日志库，性能极其优秀，与其他流行日志库的性能对比可移步[代码仓库](https://github.com/uber-go/zap)查看。<!--more-->
## 快速开始

> `Must` 是 zap 提供的一个将创建 Logger 返回值快速处理的函数，签名为 `Must(logger *Logger, err error) *Logger`，如果 `err` 为 `nil`，返回 `logger`，否则 panic

```golang
func main() {
	logger := zap.Must(zap.NewProduction())
	defer logger.Sync()

	logger.Info("Hello from Zap logger!")
}
```

有别于其他日志库，zap 没有提供默认的全局日志打印工具，在打印日志前我们必须手动创建 Logger 实例。  
虽然 zap 没有直接提供可以用的 Logger，但它预制了三套快速构建 Logger 的方法：

1. NewProduction()  
   对应生产环境，运行在 Info 日志级别，打印 Unix 秒的浮点数时间戳，日志以 `json` 格式输出，输出 caller（打印日志的代码位置）
2. NewDevelopment()  
   对应本地开发，运行在 Debug 日志级别，以一种更利于阅读的方式打印日志，可读的日志时间
3. NewExample()  
   单元测试专用，简洁无时间戳输出，json 格式，运行在 Debug 级别

通常我们使用配置文件或者环境变量方式切换 Logger：

```golang
func main() {
	logger := zap.Must(zap.NewProduction())
	switch strings.ToLower(os.Getenv("ENV")) {
	case "dev":
		logger = zap.Must(zap.NewDevelopment())
	case "test":
		logger = zap.NewExample()
	}
	// logger = zap.NewExample()
	defer logger.Sync()

	logger.Info("Hello from Zap logger!")
}
```

## 设置全局 Logger

通常我们会设置一个全局的 `Logger` 方便代码快速的完成日志输出，`zap` 提供了 `ReplaceGlobals` 来设置全局 `Logger`，使用时我们通过 `zap.L()` 获取。

> ⚠️值得注意的是，如果忘记了设置全局 `Logger`，日志将不会打印，也不会有任何报错或提示。

```golang
func main() {
	logger := zap.Must(zap.NewDevelopment())
	defer logger.Sync()

	zap.ReplaceGlobals(logger)

	zap.L().Info("using global")
}
```

## 日志等级

zap 内置了以下日志级别：

```golang
const (
	DebugLevel  Level = iota - 1 // -1 用于记录调试相关信息，辅助排查问题
	InfoLevel                    // 0 记录程序正常运行时的常规操作日志
	WarnLevel                    // 1 记录异常苗头，提示存在非正常现象，需留意，避免恶化成更严重故障
	ErrorLevel                   // 2 记录程序运行中出现的非预期错误
	DPanicLevel                  // 3 开发环境专用严重错误级别；开发环境下等同于 PANIC，生产环境降级为 ERROR 输出
	PanicLevel                   // 4 打印日志后直接触发程序 panic 崩溃
	FatalLevel                   // 5 打印日志后执行 os.Exit(1)，直接终止整个程序
)
```

## 日志打印 API

zap 提供了两套日志打印 API，第一套通过 `zap.Logger` 提供。

### zap.Logger

`zap.Logger` 为每一个日志等级提供了一个打印 API，它们的使用方法完全相同，这里仅以 Info 为例，进行演示说明：

函数签名如下：  
`func (log *zap.Logger) Info(msg string, fields ...zap.Field)`

其中参数：  
* msg：是要打印的信息
* fields：是 zap 定义的键值对抽象，通常使用 zap 内置的函数产生

```golang
func main() {
	logger := zap.Must(zap.NewProduction())
	defer logger.Sync()

	logger.Info(
		"hello",
		zap.String("server", "account"),
		zap.Bool("online", true),
	)

	// Output:
	// {"level":"info","ts":1783604497.0577993,"caller":"api/main.go:10","msg":"hello","server":"account","online":true}
}
```

zap 提供几乎所有常用类型的 field 构建函数（对于结构体使用 zap.Any 进行处理），这里就不一一列举了，需要使用时可以通过官方文档或者代码提示功能自行探索。  
通过这种强类型的构建函数，可以减少不必要的反射，减少不必要的性能损失。

### 常用 Field 构造函数速查

| 函数                       | 类型                    | 说明                       |
| -------------------------- | ----------------------- | -------------------------- |
| `zap.String(key, val)`     | string                  | 字符串                     |
| `zap.Int(key, val)`        | int                     | 整数                       |
| `zap.Int64(key, val)`      | int64                   | 64 位整数                  |
| `zap.Uint(key, val)`       | uint                    | 无符号整数                 |
| `zap.Float64(key, val)`    | float64                 | 浮点数                     |
| `zap.Bool(key, val)`       | bool                    | 布尔值                     |
| `zap.Duration(key, val)`   | time.Duration           | 时间段                     |
| `zap.Time(key, val)`       | time.Time               | 时间点                     |
| `zap.Any(key, val)`        | interface{}             | 任意类型（使用反射）       |
| `zap.Object(key, val)`     | zapcore.ObjectMarshaler | 自定义对象（需实现接口）   |
| `zap.Array(key, val)`      | zapcore.ArrayMarshaler  | 自定义数组（需实现接口）   |
| `zap.Error(err)`           | error                   | 错误，key 固定为 `"error"` |
| `zap.ByteString(key, val)` | []byte                  | 字节切片                   |
| `zap.Stringer(key, val)`   | fmt.Stringer            | 实现 Stringer 接口的类型   |
| `zap.Namespace(key)`       | —                       | 后续字段嵌套在该命名空间下 |

### 日志上下文

在生产环境中，对于一个用户的相关日志，我们往往希望在日志中携带这个用户的唯一标识，以便于对一个玩家的相关日志进行筛选。
除此之外，对于某台机器上的所有日志，某个调用链路上的日志，都有添加唯一标识进行区分的需求。为了避免重复编码，zap 提供了
With 和 WithLazy 两个方法为 Logger 添加上下文。将希望添加的字段以 zap.Field 参数形式传入后，会生成一个新的 Logger，
使用新的 Logger 打印日志会默认携带之前传入的字段信息。

With 和 WithLazy 的区别是：With 会在执行时完成取值，而 WithLazy 会在第一次打印日志或再一次连接 With 时进行取值并固化。对于序列化开销大，
并且有一定概率不会打印的内容是一种不小的性能节省。

> 注意：
> 只有 Any / Object / Array / Error / Binary 这几个 Field 使用 WithLazy 时，日志打印的是第一次取值的结果。
> 对于其他情况，都使用 WithLazy 执行时的值。

```golang
package main

import (
	"time"

	"go.uber.org/zap"
)

func main() {

	type TestStruct struct {
		TT int64
	}

	ts := new(TestStruct)
	intp := new(int)

	logger := zap.Must(zap.NewProduction())
	defer logger.Sync()

	logger = logger.With(zap.String("server", "account")).
		WithLazy(zap.Any("ts", ts), zap.Intp("intp", intp))

	ts.TT = time.Now().Unix()
	*intp = 99
	logger.Info("user login")
	time.Sleep(time.Second)
	ts.TT = time.Now().Unix()
	logger.Info("user logout")
}

// Output:
// {"level":"info","ts":1783693404.481155,"caller":"with/main.go:26","msg":"user login","server":"account","ts":{"TT":1783693404},"intp":0}
// {"level":"info","ts":1783693405.481245,"caller":"with/main.go:29","msg":"user logout","server":"account","ts":{"TT":1783693404},"intp":0}
```

### Named 与 AddCallerSkip

`Named` 为 Logger 添加一个命名前缀，日志输出中会携带该名称，便于按模块区分日志来源：

```golang
logger := zap.Must(zap.NewProduction()).Named("http")
logger.Info("request received")
// 输出中 caller 会显示类似 "http/main.go:10"
```

`AddCallerSkip` 用于跳过调用栈层级。当我们封装日志函数时，caller 会指向封装函数而非真正的调用处，通过 `AddCallerSkip(1)` 可以修正：

```golang
func main() {
	logger := zap.Must(zap.NewDevelopment()).With(zap.String("app", "demo"))

	// 不加 AddCallerSkip，caller 会指向 logInfo 函数内部
	// 加了之后，caller 指向真正的调用处 main
	loggerWithSkip := logger.WithOptions(zap.AddCallerSkip(1))
	logInfo(loggerWithSkip, "server started")
}

func logInfo(logger *zap.Logger, msg string) {
	logger.Info(msg)
}
```

### zap.Sugar

除了使用 zap.Logger 提供的高效日志打印 API 外，zap 通过 zap.Sugar 提供了另一套更加方便的日志打印 API，虽然它会牺牲一小部分性能。  
Sugar 为每一个日志级别提供了四种打印 API，对于每一种级别，它们仅仅是通过前缀进行区别，这里直接使用 Info 级别进行演示说明：  

```golang
func main() {
	logger := zap.Must(zap.NewProduction())
	defer logger.Sync()
	sugar := logger.Sugar()

	sugar.Info("fmt.Print")
	sugar.Infof("fmt.Printf")
	sugar.Infoln("fmt.Println")
	sugar.Infow("msg", "k1", "v1", "k2", "v2")
}

// {"level":"info","ts":1783605280.1782732,"caller":"sugar/main.go:12","msg":"fmt.Print"}
// {"level":"info","ts":1783605280.178496,"caller":"sugar/main.go:13","msg":"fmt.Printf"}
// {"level":"info","ts":1783605280.1785016,"caller":"sugar/main.go:14","msg":"fmt.Println"}
// {"level":"info","ts":1783605280.178506,"caller":"sugar/main.go:15","msg":"msg","k1":"v1","k2":"v2"}
```
对于 Info, Infof, Infoln 可以类比 fmt 提供的三种打印方法。  
对于 Infow，它的参数由一个 string 类型的 message 和可变的 key-value 对构成。key-value 中的 key 必须是字符串。

> ⚠️ 对于 Infow 这类打印 API，如果 key-value 键值对异常（如 key-value 数量不匹配，key 不是字符串）  
> 在 Production 环境下会额外打印一条报错信息
> 在 Development 环境下会直接 panic

zap.Sugar 也支持 With 和 WithLazy 添加上下文，它的参数 `...interface{}`，用户可以混合填入 key-value 对和 zap.Field。
其他的功能效果和 Logger 的类似，这里就不展开了。

## 相互转换

zap.Logger 和 zap.Sugar 可以方便的进行相互转换，而且几乎没有开销，所以可以在希望的任何时候进行转换：

```golang
func main() {
	logger := zap.Must(zap.NewProduction())
	defer logger.Sync()

	sugar := logger.Sugar()
	logger2 := sugar.Desugar()

	logger2.Info("test")
}
```

## 自定义 Logger

zap 自定义 Logger 有多种方式，这里主要记录两种：

1. 轻度使用 zap.Config

	这种方式可以直接参考 zap.NewProduction 的实现

	```golang
	...
	func NewProduction(options ...Option) (*Logger, error) {
		return NewProductionConfig().Build(options...)
	}
	...
	func NewProductionConfig() Config {
		return Config{
			Level:       NewAtomicLevelAt(InfoLevel), // 日志等级
			Development: false, // 是否为开发模式 会修改 DPanic 级别日志的处理 堆栈的信息的详细程度
			Sampling: &SamplingConfig{ // 采样
				Initial:    100,
				Thereafter: 100,
			},
			Encoding:         "json", // 编码器，支持注册自定义编码器
			EncoderConfig:    NewProductionEncoderConfig(), // 编码器配置
			OutputPaths:      []string{"stderr"}, // 输出路径
			ErrorOutputPaths: []string{"stderr"}, // 错误输出路径
		}
	}
	...
	```
	这种方式适用于对定制化要求不高的场景，比如仅希望调整日志级别，日志输出位置等的情况。
2. 生产使用

	这里直接上代码了，说明内容添加在注释上了
	```golang
	package main

	import (
		"os"

		"go.uber.org/zap"
		"go.uber.org/zap/zapcore"
	)

	func createLogger() *zap.Logger {
		// 日志存放文件
		file, _ := os.OpenFile("./test.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
		// 日志等级
		level := zap.NewAtomicLevelAt(zap.InfoLevel)
		// 利用预制的配置创建编码器
		productionCfg := zap.NewProductionEncoderConfig()
		productionCfg.TimeKey = "timestamp" // 修改时间 key
		productionCfg.EncodeTime = zapcore.ISO8601TimeEncoder
		fileEncoder := zapcore.NewJSONEncoder(productionCfg)
		// 创建 zap.Core
		core := zapcore.NewCore(fileEncoder, file, level)
		// 创建 logger
		return zap.New(core)
	}

	func main() {
		logger := createLogger()
		defer logger.Sync()
		logger.Info("Hello from Zap!")
	}
	```
	这种方式才是真正能使用到生产上的，接下来对其中的一些功能点进行详细演示。

### AtomicLevel

在第二种自定义 Logger 方法中，我们使用 zap.NewAtomicLevelAt 创建了日志等级配置，它的返回值是 zap.AtomicLevel。  
AtomicLevel 有两个很有用的方法：SetLevel 和 Enabled。

SetLevel 允许我们动态的设置日志等级，可以通过一个 API 接口为我们的程序提供调整日志等级的能力。
```golang
level.SetLevel(zap.DebugLevel)
```

Enabled 允许我们在打印日志前判断对应等级的日志是否会打印，避免一些不必要的日志内容计算消耗。
```golang
if level.Enabled(zap.InfoLevel) {
	logger.Info("xxx")
}
```

### Sampling

Sampling 采样是用来对日志削峰的。采用 Level + Msg 进行分组采样，控制每秒内前 Initial 条无条件打印，超过部分每 Thereafter 条打印一条。Hook 用来配置采样判定后的处理。  
主要作用是拦截大量重复报错 / 相同业务日志，避免海量刷屏日志打满磁盘、阻塞 IO、抬高日志存储成本。

```golang
type SamplingConfig struct {
	Initial    int                                           `json:"initial" yaml:"initial"`
	Thereafter int                                           `json:"thereafter" yaml:"thereafter"`
	Hook       func(zapcore.Entry, zapcore.SamplingDecision) `json:"-" yaml:"-"`
}
```

### 日志轮转

zap 没有直接提供日志轮转功能，但我们可以使用 gopkg.in/natefinch/lumberjack.v2 创建带轮转能力的 SyncWriter：

```golang
...
file := zapcore.AddSync(&lumberjack.Logger{
	Filename:   "./app.log", // 日志文件路径
	MaxSize:    100,         // 单文件最大MB，超过切割
	MaxBackups: 10,          // 最多保留多少个归档文件
	MaxAge:     7,           // 旧日志保留天数
	Compress:   true,        // 是否压缩归档日志
	LocalTime:  true,        // 归档文件名使用本地时间（默认UTC）
})
...
```

### zapcore.Core

`zapcore.Core` 是 zap 的核心抽象接口，定义了日志处理的三个关键行为：检查等级、写入日志、同步缓冲区。自定义 Core 可以实现完全自定义的日志处理逻辑，比如写入数据库、过滤特定字段、添加全局标签等。

```golang
type Core interface {
	// Enabled 检查给定等级的日志是否会被处理
	Enabled(Level) bool
	// With 添加一组 Core 会持有的字段
	With([]Field) Core
	// Check 判断是否应该记录这条日志，返回 Entry 加上可能修改后的 Core
	Check(Entry, *CheckedEntry) *CheckedEntry
	// Write 将日志条目和字段写入底层输出
	Write(Entry, []Field) error
	// Sync 刷新任何缓冲的日志数据
	Sync() error
}
```

一个简单的示例 — 自定义 Core 实现日志写入到标准输出，同时过滤掉 Debug 级别：

```golang
func main() {
	core := zapcore.NewCore(
		zapcore.NewConsoleEncoder(zap.NewDevelopmentEncoderConfig()),
		zapcore.AddSync(os.Stdout),
		zap.InfoLevel, // 只处理 Info 及以上级别
	)
	logger := zap.New(core)
	defer logger.Sync()

	logger.Debug("这条不会输出")
	logger.Info("这条会输出")
}
```

## 多路输出

这里提供两种方法：

如果只是想要在文件和终端等地方同时打印，可以使用 io.MultiWriter：

```golang
...
file, _ := os.OpenFile("./test.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
writer := io.MultiWriter(os.Stderr, file)
writeSyncer := zapcore.AddSync(writer)
...
```

如果有更多定制化需求，如不同路径打印不同的信息，不同的日志等级等，可以使用 zapcore.NewTee：

```golang
func createLogger() *zap.Logger {
	stdout := zapcore.AddSync(os.Stdout)

	file := zapcore.AddSync(&lumberjack.Logger{
		Filename:   "logs/app.log",
		MaxSize:    10, // megabytes
		MaxBackups: 3,
		MaxAge:     7, // days
	})

	level := zap.NewAtomicLevelAt(zap.InfoLevel)

	productionCfg := zap.NewProductionEncoderConfig()
	productionCfg.TimeKey = "timestamp"
	productionCfg.EncodeTime = zapcore.ISO8601TimeEncoder

	developmentCfg := zap.NewDevelopmentEncoderConfig()
	developmentCfg.EncodeLevel = zapcore.CapitalColorLevelEncoder

	consoleEncoder := zapcore.NewConsoleEncoder(developmentCfg)
	fileEncoder := zapcore.NewJSONEncoder(productionCfg)

	core := zapcore.NewTee(
		zapcore.NewCore(consoleEncoder, stdout, level),
		zapcore.NewCore(fileEncoder, file, level),
	)

	return zap.New(core)
}
```

## ObjectMarshaler

在生产环境中隐藏用户敏感信息是必不可少的，在 zap 中可以通过为类型实现 ObjectMarshaler 接口实现：

```golang
type ObjectMarshaler interface {
	MarshalLogObject(ObjectEncoder) error
}
```

```golang
package main

import (
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

// 自定义业务结构体
type User struct {
	ID       int64
	Name     string
	Password string // 敏感字段，不打印
}

// 实现 zapcore.ObjectMarshaler 接口
func (u User) MarshalLogObject(enc zapcore.ObjectEncoder) error {
	enc.AddInt64("id", u.ID)
	enc.AddString("username", u.Name)
	// 不添加 Password，脱敏隐藏
	return nil
}

func main() {
	logger, _ := zap.NewProduction()
	defer logger.Sync()

	user := User{ID: 1001, Name: "zhangsan", Password: "123456"}
	logger.Info("user login", zap.Object("user_info", user))
}
```

除了隐藏敏感信息外，还能自定义字段名称，减少不少使用 zap.Any 带来的反射消耗。

## 最佳实践与常见陷阱

### 务必调用 logger.Sync()

`Sync()` 会将缓冲区中的日志数据刷入底层输出。如果不调用，程序异常退出时最后几条日志可能丢失。通常配合 `defer` 使用：

```golang
logger := zap.Must(zap.NewProduction())
defer logger.Sync() // 确保所有日志被写入
```

### 避免在热路径中重复创建 Logger

Logger 创建有一定开销，应在程序初始化时创建并复用。不要在循环或高频调用的函数中反复创建：

```golang
// 错误：每次调用都创建新 Logger
func handleRequest() {
	logger := zap.Must(zap.NewProduction())
	logger.Info("handling request")
}

// 正确：使用全局 Logger 或通过依赖注入传递
func handleRequest(logger *zap.Logger) {
	logger.Info("handling request")
}
```

### zap.Any vs 强类型 Field

`zap.Any` 虽然方便，但内部通过反射序列化值，性能不如强类型构造函数。在性能敏感的场景中，优先使用 `zap.String`、`zap.Int` 等：

```golang
// 反射方式，有额外开销
logger.Info("user login", zap.Any("user", user))

// 强类型方式，零反射
logger.Info("user login",
	zap.Int64("user_id", user.ID),
	zap.String("username", user.Name),
)
```

### 结构化日志字段命名规范

保持字段名风格一致有助于日志检索和分析。推荐使用 `snake_case`，这也是 JSON 社区和大多数日志系统的惯例：

```golang
// 推荐
logger.Info("user login",
	zap.String("user_id", "12345"),
	zap.String("request_path", "/api/login"),
)

// 不推荐混用驼峰
logger.Info("user login",
	zap.String("userId", "12345"),
	zap.String("requestPath", "/api/login"),
)
```

## 作为 slog 后端

通过将 zap 作为 slog 的后端，可以利用 slog 的标准化（快速切换后端日志库的能力），同时兼顾 zap 高性能，在社区内是一个很受欢迎的方案。  
zap 官方提供了另一个模块来支持这一方案：`go.uber.org/zap/exp/zapslog`，使用方法如下：

```golang
package main

import (
	"log/slog"

	"go.uber.org/zap"
	"go.uber.org/zap/exp/zapslog"
)

func main() {
	zlogger := zap.Must(zap.NewProduction())

	defer zlogger.Sync()

	slogger := slog.New(zapslog.NewHandler(zlogger.Core()))

	slogger.Info(
		"receive request",
		slog.String("method", "GET"),
		slog.String("path", "/api/user"),
		slog.Int("status", 200),
	)
}
```

## 参考资料

[https://github.com/uber-go/zap](https://github.com/uber-go/zap)  
[https://pkg.go.dev/go.uber.org/zap](https://pkg.go.dev/go.uber.org/zap)  
[A Comprehensive Guide to Zap Logging in Go](https://betterstack.com/community/guides/logging/go/zap/)  
[A Practitioner's Guide to Logging in Go with Zap](https://www.dash0.com/guides/logging-in-go-with-zap)  
[从Go log库到Zap，怎么打造出好用又实用的Logger](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247493846&idx=1&sn=de3eee6b0d1a02bcd966ea07b7401c99&chksm=fa833941cdf4b057d45b7cbf2748abb1c1e79d0572b55705f868f35581c8d9ed3d9812f8e27e&cur_album_id=1326949382503219201&scene=189)  
