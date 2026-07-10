---
date: "2021-04-17T00:00:00+08:00"
title: Golang -- 协程池设计与实践
tags: ["Golang", "goroutine", "concurrent"]
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

goroutine 是一种轻量级的并发处理方案，是 Go 语言区别于其他语言的主要特点之一。goroutine 默认栈大小为 8KB（支持动态增长），官方宣称使用 Go 语言可以轻松的在单机上启动 10 万个协程。然而在实际使用中并不推荐这样使用。

如果对 GMP 调度模型有所了解的话，就会知道盲目的开启协程不仅不会带来性能的提升，反而会将性能浪费在协程调度上。此外，极端情况下过度使用协程也会造成 OOM 的情况。故而，我们往往会对协程的数量进行限制。

**协程数量建议：**

- **CPU 密集型任务**：协程数量 ≈ 逻辑 CPU 核心数
- **IO 密集型任务**：协程数量可以远大于 CPU 核心数（通常为 CPU 核心数的 10-100 倍）

下文将采用斐波那契数列计算作为示例，演示几种协程数量控制的方法。注意：代码实现本身并不重要。

斐波那契数的计算采用如下方法：

```go
func Fibo(n int) int {
    if n <= 1 {
        return n
    }
    return Fibo(n-1) + Fibo(n-2)
}
```

## 简单的数量控制

### 示例代码

```go
// 计算 0-n 的斐波那契数
func GetFiboList(n int) []int {
    // 确定计算协程数量
    numGoroutine := 3

    res := make([]int, n+1)
    nChan := make(chan int, 10)
    fChan := make(chan struct{}, 10)

    // 启动一个协程向管道中写入要计算的数
    go func() {
        for i := 0; i <= n; i++ {
            nChan <- i
        }
    }()

    // 启动工作协程
    for i := 0; i < numGoroutine; i++ {
        go func() {
            for num := range nChan {
                res[num] = Fibo(num)
                fChan <- struct{}{}
            }
        }()
    }

    // 等待任务完成
    for i := 0; i <= n; i++ {
        <-fChan
    }
    return res
}
```

### 特点说明

该方法不对协程进行长期保持，在完成任务后协程会自动销毁，可以理解为一次性使用的协程。

**方案优势：**

- 可以根据具体情况（如：问题规模、逻辑 CPU 数量、已有协程数量等）对协程数量进行限制，从而实现性能的最大利用。
- 任务完成后协程会自动销毁，不会长时间占用资源。

**适用场景：**

- 任务数量已知且固定
- 需要批量处理一批任务
- 不需要长期维护协程

## 简单的协程池

### 示例代码

```go
type Task struct {
    f func() error
}

func (t *Task) Execute() error {
    return t.f()
}

type Pool struct {
    EntryChannel chan *Task // 任务入口
    JobChannel   chan *Task // 内部任务队列
    workerNum    int        // 最大协程数量
}

func NewPool(gnum int) *Pool {
    return &Pool{
        EntryChannel: make(chan *Task, 10),
        JobChannel:   make(chan *Task, 10),
        workerNum:    gnum,
    }
}

func (p *Pool) worker(workerId int) {
    for task := range p.JobChannel {
        err := task.Execute()
        if err != nil {
            fmt.Printf("worker %d 执行出错: %v\n", workerId, err)
        } else {
            fmt.Println("worker", workerId, "执行完毕")
        }
    }
}

func (p *Pool) Run() {
    for i := 0; i < p.workerNum; i++ {
        go p.worker(i)
    }

    go func() {
        for task := range p.EntryChannel {
            p.JobChannel <- task
        }
    }()
}

func main() {
    p := NewPool(3)
    p.Run()

    for i := 0; i < 20; i++ {
        // 使用立即执行的匿名函数来捕获当前的 i 值
        f := func(n int) func() error {
            return func() error {
                Work(n)
                return nil
            }
        }(i)

        p.EntryChannel <- &Task{f}
        time.Sleep(time.Second)
    }
}

func Work(n int) {
    fmt.Printf("fibo(%d)=%d\n", n, Fibo(n))
}
```

### 特点说明

此方案创建的协程为持久驻留协程，只要有新的任务写入，协程池内协程就会获取任务并执行。

**方案优势：**

- 一次创建协程池后工作协程会长期驻留，只需要在需要使用时向任务入口写入任务，协程池内协程就会自动工作起来。
- 协程工作内容不受限制，只要按照任务内容进行封装即可写入协程池。可以方便的应对不同的任务。
- 可以修改任务结构体，构建复杂的任务调度机制（如优先级队列等），从而满足复杂的应用需求。

**适用场景：**

- 任务数量不确定或持续产生
- 需要长期维护协程池
- 需要处理多种类型的任务

## 一些协程池推荐

### go-playground/pool

go-playground/pool 相比之前简单的协程池，对 pool、worker 的状态有了很好的管理。

**存在的问题：**

- 在第一个实现的简单 goroutine 池和 go-playground/pool 中，都是先启动预定好的 goroutine 来完成任务执行
- 在并发量远小于任务量的情况下确实能够做到 goroutine 的复用
- 如果任务量不多则会导致任务分配到每个 goroutine 不均匀，甚至可能出现启动的 goroutine 根本不会执行任务从而导致浪费
- 对于协程池也没有动态的扩容和缩小

### ants

ants 是一个受 fasthttp 启发的高性能协程池，fasthttp 在特定场景下号称比 Go 原生的 net/http 快数倍，其快速高性能的原因之一就是采用了各种池化技术。

**工作模型：**

- ants 相比之前两种协程池，其模型更像是之前接触到的数据库连接池
- 需要从空余的 worker 中取出一个来执行任务，当无可用空余 worker 的时候再去创建
- 当 pool 的容量达到上限之后，剩余的任务阻塞等待当前进行中的 worker 执行完毕将 worker 放回 pool，直至 pool 中有空闲 worker

**内存管理：**

- 定期清除过期 worker（一定时间内没有分配到任务的 worker）
- 实现了一种适用于大批量相同任务的 pool，这种 pool 与一个需要大批量重复执行的函数绑定，避免了调用方不停的创建，更加节省内存

## 方案对比

| 特性         | 简单的数量控制 | 简单的协程池 | go-playground/pool | ants                     |
| ------------ | -------------- | ------------ | ------------------ | ------------------------ |
| 协程生命周期 | 一次性使用     | 长期驻留     | 长期驻留           | 长期驻留（支持过期清理） |
| 动态扩缩容   | 不支持         | 不支持       | 不支持             | 支持                     |
| 内存管理     | 自动回收       | 需手动管理   | 较好               | 优秀                     |
| 适用场景     | 批量固定任务   | 通用场景     | 通用场景           | 高性能场景               |
| 复杂度       | 低             | 中           | 中                 | 高                       |

**选择建议：**

- **简单批量任务**：使用"简单的数量控制"方案
- **通用长期任务**：使用"简单的协程池"或 go-playground/pool
- **高性能生产环境**：推荐使用 ants

## 参考资料

- https://studygolang.com/articles/15477
- https://github.com/go-playground/pool
- https://segmentfault.com/a/1190000018193161
