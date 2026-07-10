---
date: "2021-05-11T00:00:00+08:00"
title: "Golang -- 空结构体"
tags: ["Golang", "优化"]
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

Golang 正常的 struct 就是普通的一个内存块，必定是占用一小块内存的，并且结构体的大小是要经过边界和长度对齐的。然而，对于 struct{}，Golang 中的空结构体，却有着自己独特的特性。

## 空结构体的大小

在 Golang 中我们常使用 unsafe.Sizeof 查看变量的字节大小。如下代码使用 unsafe.Sizeof 查看空 struct 的大小。

```go
import (
    "fmt"
    "unsafe"
)

func main() {
    fmt.Println(unsafe.Sizeof(struct{}{}))
}
```

结果如下：

```
> go run ./main.go
> 0
```

也就是说空结构体是不占空间的。

原理说明：空结构体内存分配时都会指向一个 zerobase 的 8 字节 uintptr 型全局变量，可以创建多个空结构体打印地址来验证，会发现它们有相同的地址。

## 空结构体的作用

在 Golang 中即使是布尔型变量也会占用一个字节的空间，基于空结构体不占用存储空间的特性，我们往往用它来节省内存空间（或者说空结构体设计的初衷就是节省空间）。

### 用在不传递数据的通道中

```go
import "fmt"

func worker(ch chan struct{}) {
    <-ch
    fmt.Println("do something")
    close(ch)
}

func main() {
    ch := make(chan struct{})
    go worker(ch)
    ch <- struct{}{}
}
```

### 用作并发控制（信号量）

通过带缓冲的 `chan struct{}` 可以实现简单的并发控制，限制同时运行的 goroutine 数量：

```go
import (
    "fmt"
    "sync"
    "time"
)

func main() {
    sem := make(chan struct{}, 3) // 最多 3 个并发
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        sem <- struct{}{} // 获取令牌，满时阻塞
        go func(id int) {
            defer func() {
                <-sem // 释放令牌
                wg.Done()
            }()
            fmt.Println("worker", id, "start")
            time.Sleep(time.Second)
            fmt.Println("worker", id, "done")
        }(i)
    }

    wg.Wait()
}
```

### 用在 map 中

可以用在 map 中，利用 map 模拟集合。

```go
import "fmt"

type Set map[string]struct{}

func (s Set) Has(key string) bool {
    _, ok := s[key]
    return ok
}

func (s Set) Add(key string) {
    s[key] = struct{}{}
}

func (s Set) Delete(key string) {
    delete(s, key)
}

func main() {
    s := make(Set)
    s.Add("Tom")
    s.Add("Sam")
    fmt.Println(s.Has("Tom"))
    fmt.Println(s.Has("Jack"))
}
```

### 用作方法接收者

可以将相关方法绑定到空结构体上，作为逻辑分组的组织方式。

```go
type Door struct{}

func (d Door) Open() {
    fmt.Println("Open the door")
}

func (d Door) Close() {
    fmt.Println("Close the door")
}
```

### 作为编译期接口检查

利用空结构体，可以在编译期验证某个类型是否实现了指定接口，避免运行时才发现问题。

```go
type Speaker interface {
    Speak() string
}

type Dog struct{}

func (d Dog) Speak() string {
    return "Woof"
}

// 编译期检查：如果 Dog 没有实现 Speaker，编译器会报错
var _ Speaker = Dog{}
```

### 作为标记字段

空结构体可以作为结构体中的标记字段，用于区分不同类型，类似 Java 中的 Marker Interface。

```go
type Event struct {
    _       struct{} // 标记这是一个事件
    Payload string
}
```

### 作为 context 的私有键

用未导出的空结构体类型作为 `context.WithValue` 的 key，可以避免不同包之间的 key 冲突：

```go
import "context"

// 未导出的类型，外部包无法伪造
type contextKey struct{}

var userKey = contextKey{}

func WithUser(ctx context.Context, name string) context.Context {
    return context.WithValue(ctx, userKey, name)
}

func UserFrom(ctx context.Context) (string, bool) {
    name, ok := ctx.Value(userKey).(string)
    return name, ok
}
```

如果直接用 `string` 或 `int` 做 key，不同包可能会意外冲突。用空结构体类型作为 key 是 Go 官方推荐的做法。

## 一段神奇的代码

下面这段代码展示了空结构体在结构体中的位置对内存对齐的影响：

```go
import (
    "fmt"
    "unsafe"
)

type Person struct {
    Name string
    Age  int
}

type PersonI struct {
    _    struct{}
    Name string
    Age  int
}

type PersonII struct {
    Name string
    Age  int
    _    struct{}
}

func main() {
    fmt.Println(unsafe.Sizeof(Person{}))
    fmt.Println(unsafe.Sizeof(PersonI{}))
    fmt.Println(unsafe.Sizeof(PersonII{}))
}
```

输出结果如下：

```
> go run ./main.go
> 24
> 24
> 32
```

原理说明：空结构体作为结构体最后一个字段时，会进行填充，填充大小为上一个字段大小。结构体大小按照内存对齐条件正常计算即可。

## 性能对比

使用 `struct{}` 和 `bool` 作为 map value 的内存差异：

```go
import (
    "fmt"
    "unsafe"
)

func main() {
    boolMap := make(map[string]bool)
    structMap := make(map[string]struct{})

    for i := 0; i < 1000000; i++ {
        key := fmt.Sprintf("key_%d", i)
        boolMap[key] = true
        structMap[key] = struct{}{}
    }

    fmt.Println("bool map:", unsafe.Sizeof(boolMap))
    fmt.Println("struct{} map:", unsafe.Sizeof(structMap))
}
```

`unsafe.Sizeof` 只反映变量本身的大小（都是 map header，均为 8 字节），无法体现 value 的差异。要观察实际内存占用需要借助 `runtime.MemStats`。在百万级数据量下，`struct{}` 版本比 `bool` 版本可节省约 1MB 内存（每个 key 少 1 字节 value）。

## 标准库中的应用

Go 标准库中大量使用了空结构体：

**context.Done()**

`context.Context` 的 `Done()` 方法返回的就是 `chan struct{}`，用于通知 goroutine 退出：

```go
import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("cancelled:", ctx.Err())
            return
        default:
            // do work
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go worker(ctx)
    time.Sleep(time.Second)
    cancel()
    time.Sleep(time.Second)
}
```

**sync.WaitGroup**

`WaitGroup` 内部通过 atomic 和信号量实现同步计数，无需传递额外数据。

**http.CloseNotifier（已废弃）**

早期的 `http.CloseNotifier` 接口同样使用 `chan struct{}` 作为连接关闭的通知通道。

## 底层原理

空结构体在 Go 运行时有一个特殊的全局变量 `runtime.zerobase`：

```go
// src/runtime/malloc.go
var zerobase uintptr
```

所有空结构体实例的地址都指向这个 `zerobase`，它是一个 8 字节的全局变量，本身不占用户内存。当分配 `struct{}` 时，Go 的内存分配器直接返回 `zerobase` 的地址，而不是真正分配内存。

可以通过以下代码验证：

```go
import "fmt"

func main() {
    a := struct{}{}
    b := struct{}{}
    fmt.Printf("a: %p\n", &a)
    fmt.Printf("b: %p\n", &b)
    // 输出相同的地址
}
```

## 参考资料

- [Go 语言高性能编程](https://geektutu.com/post/hpg-empty-struct.html)
- [空结构体是什么](https://blog.csdn.net/qiya2007/article/details/111502649)
