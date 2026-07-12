---
date: "2024-05-26T18:09:59+08:00"
title: "Golang -- metrics 参考"
tags: ["Golang", "metrics"]
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

metrics 包提供了一个稳定的接口来访问由 Go 运行时暴露的预定义好的指标数据。功能类似于 `runtime.ReadMemStats` 和 `runtime/debug.ReadGCStats`，但是更加通用。<!--more-->

# 版本要求

`runtime/metrics` 从 **Go 1.16** 开始引入，使用前请确认项目 Go 版本 >= 1.16。

# 与 ReadMemStats 的区别

| | `runtime/metrics` | `runtime.ReadMemStats` |
|---|---|---|
| 稳定性 | 官方承诺接口稳定，不会破坏性变更 | 可能在版本间变动 |
| 命名方式 | 字符串路径（如 `/gc/heap/goal:bytes`） | 结构体字段（如 `HeapGoal`） |
| 数据类型 | 支持 uint64、float64、Float64Histogram | 仅 uint64、uint32 等整数类型 |
| 性能 | 按需采样，只读取需要的指标；`metrics.Read` 不产生堆分配 | 一次性填充整个 MemStats 结构体，每次调用都会分配 |
| 粒度 | 更细，提供 CPU 时间分类、调度延迟分布等 | 较粗，主要关注内存和 GC |

**选择建议**：新项目优先使用 `runtime/metrics`；如果只需要简单的内存统计且不需要 histogram 数据，`ReadMemStats` 仍然可用。

# Interface

metrics 被定义为字符串，而不是一个结构体的字段，这更有利于拓展。完整的 metrics 列表可以查看由 `All` 方法返回的 `Description` 列表。

```go
package main

import (
	"fmt"
	"runtime/metrics"
)

func main() {
	desc := metrics.All()
	fmt.Println("total:", len(desc))
	for _, m := range desc {
		fmt.Println(
			"\nName:", m.Name,
			"\nKind:", m.Kind, // 数据类型
			// "\nDesc:", m.Description,
			// "\nCumulative:", m.Cumulative, // 是否累计
		)
	}
}
```

```sh
total: 81

Name: /cgo/go-to-c-calls:calls

Name: /cpu/classes/gc/mark/assist:cpu-seconds

Name: /cpu/classes/gc/mark/dedicated:cpu-seconds

Name: /cpu/classes/gc/mark/idle:cpu-seconds

Name: /cpu/classes/gc/pause:cpu-seconds
...
# 总表放在后面了，按需取用。
```

其中 Kind 表示这个指标所使用的数据类型，Go 官方保证不会对这个类型进行修改。目前有以下几种类型:

```go
const (
	// KindBad indicates that the Value has no type and should not be used.
	KindBad ValueKind = iota

	// KindUint64 indicates that the type of the Value is a uint64.
	KindUint64

	// KindFloat64 indicates that the type of the Value is a float64.
	KindFloat64

	// KindFloat64Histogram indicates that the type of the Value is a *Float64Histogram.
	KindFloat64Histogram
)
```

Kind 值与类型的对应关系：

| Kind 值 | 类型 | 说明 |
|---------|------|------|
| 0 | KindBad | 无效类型，不应使用 |
| 1 | KindUint64 | uint64 整数 |
| 2 | KindFloat64 | float64 浮点数 |
| 3 | KindFloat64Histogram | *Float64Histogram 直方图 |

# Cumulative 字段

部分指标的 `Cumulative: true` 表示该值是**单调递增的累计值**（如 "累计堆内存分配"）。要得到某个时间窗口内的速率，需要做差：

```go
package main

import (
	"fmt"
	"runtime/metrics"
	"time"
)

func main() {
	const myMetric = "/gc/heap/allocs:bytes"
	samples := []metrics.Sample{{Name: myMetric}}

	// 第一次采样
	metrics.Read(samples)
	alloc1 := samples[0].Value.Uint64()

	// 等待一段时间后再次采样
	time.Sleep(time.Second)
	metrics.Read(samples)
	alloc2 := samples[0].Value.Uint64()

	// 计算速率
	allocRate := alloc2 - alloc1
	fmt.Printf("分配速率: %d bytes/s\n", allocRate)
}
```

`Cumulative: false` 的指标（如 "活跃堆内存"）是瞬时值，直接使用即可。

# Float64Histogram 的使用

当 `Kind == KindFloat64Histogram` 时，值类型为 `*metrics.Float64Histogram`，包含 buckets 和 counts：

```go
package main

import (
	"fmt"
	"runtime/metrics"
)

func main() {
	samples := []metrics.Sample{{Name: "/sched/latencies:seconds"}}
	metrics.Read(samples)

	if samples[0].Value.Kind() == metrics.KindFloat64Histogram {
		hist := samples[0].Value.Float64Histogram()
		fmt.Println("Bucket boundaries:", hist.Buckets)
		fmt.Println("Bucket counts:", hist.Counts)
		// Counts[i] 表示值落在 [Buckets[i], Buckets[i+1]) 区间内的次数
		// Counts[0] 是 (-inf, Buckets[0])
		// Counts[len(Buckets)] 是 [Buckets[len-1], +inf)
	}
}
```

典型应用：分析调度延迟分布、GC 暂停时间分布等。

# 采样

## 基础用法

```go
package main

import (
	"fmt"
	"runtime/metrics"
)

func main() {
	const myMetric = "/memory/classes/heap/free:bytes"

	// 需要哪些数据就添加哪些采样需求
	sample := make([]metrics.Sample, 1)
	sample[0].Name = myMetric

	// 采样
	metrics.Read(sample)

	if sample[0].Value.Kind() == metrics.KindBad {
		panic(fmt.Sprintf("metric %q no longer supported", myMetric))
	}

	freeBytes := sample[0].Value.Uint64()

	fmt.Printf("free but not released memory: %d\n", freeBytes)
}
```

## 周期性采样

```go
package main

import (
	"fmt"
	"runtime/metrics"
	"time"
)

func main() {
	// 同时采样多个指标
	samples := []metrics.Sample{
		{Name: "/gc/heap/allocs:bytes"},
		{Name: "/gc/cycles/total:gc-cycles"},
		{Name: "/sched/goroutines:goroutines"},
	}

	var prevAlloc uint64
	var prevCycles uint64
	first := true

	for range time.Tick(5 * time.Second) {
		metrics.Read(samples)

		alloc := samples[0].Value.Uint64()
		cycles := samples[1].Value.Uint64()
		goroutines := samples[2].Value.Uint64()

		if !first {
			allocRate := (alloc - prevAlloc) / 5
			fmt.Printf("alloc rate: %d bytes/s, GC cycles: %d, goroutines: %d\n",
				allocRate, cycles-prevCycles, goroutines)
		}

		prevAlloc = alloc
		prevCycles = cycles
		first = false
	}
}
```

# 指标信息

## CGO

### GO 调用 C 次数

Name: /cgo/go-to-c-calls:calls
Kind: 1
Desc: Count of calls made from Go to C by the current process.
Cumulative: true

## CPU

### 协助 GC 的 cpu 时间

Name: /cpu/classes/gc/mark/assist:cpu-seconds
Kind: 2
Desc: Estimated total CPU time goroutines spent performing GC tasks to assist the GC and prevent it from falling behind the application. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### 专注 GC 的 cpu 时间

Name: /cpu/classes/gc/mark/dedicated:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent performing GC tasks on processors (as defined by GOMAXPROCS) dedicated to those tasks. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### 空闲 CPU 参与 GC 的时间

Name: /cpu/classes/gc/mark/idle:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent performing GC tasks on spare CPU resources that the Go scheduler could not otherwise find a use for. This should be subtracted from the total GC CPU time to obtain a measure of compulsory GC CPU time. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### 任务暂停时 GC 所用 cpu 时间

Name: /cpu/classes/gc/pause:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent with the application paused by the GC. Even if only one thread is running during the pause, this is computed as GOMAXPROCS times the pause latency because nothing else can be executing. This is the exact sum of samples in /sched/pauses/total/gc:seconds if each sample is multiplied by GOMAXPROCS at the time it is taken. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### GC 总时长

Name: /cpu/classes/gc/total:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent performing GC tasks. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics. Sum of all metrics in /cpu/classes/gc.
Cumulative: true

### CPU 空闲时间

Name: /cpu/classes/idle:cpu-seconds
Kind: 2
Desc: Estimated total available CPU time not spent executing any Go or Go runtime code. In other words, the part of /cpu/classes/total:cpu-seconds that was unused. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### 协助紧急内存释放 cpu 时间

Name: /cpu/classes/scavenge/assist:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent returning unused memory to the underlying platform eagerly in response to memory pressure. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### 后台任务期间释放内存 cpu 时间

Name: /cpu/classes/scavenge/background:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent performing background tasks to return unused memory to the underlying platform. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

### 释放内存总时间

Name: /cpu/classes/scavenge/total:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent performing tasks that return unused memory to the underlying platform. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics. Sum of all metrics in /cpu/classes/scavenge.
Cumulative: true

### 总 cpu 可用时间

Name: /cpu/classes/total:cpu-seconds
Kind: 2
Desc: Estimated total available CPU time for user Go code or the Go runtime, as defined by GOMAXPROCS. In other words, GOMAXPROCS integrated over the wall-clock duration this process has been executing for. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics. Sum of all metrics in /cpu/classes.
Cumulative: true

### 总用户 cpu 时间

Name: /cpu/classes/user:cpu-seconds
Kind: 2
Desc: Estimated total CPU time spent running user Go code. This may also include some small amount of time spent in the Go runtime. This metric is an overestimate, and not directly comparable to system CPU time measurements. Compare only with other /cpu/classes metrics.
Cumulative: true

## GC 循环

### 自动 GC 次数

Name: /gc/cycles/automatic:gc-cycles
Kind: 1
Desc: Count of completed GC cycles generated by the Go runtime.
Cumulative: true

### 主动 GC 次数

Name: /gc/cycles/forced:gc-cycles
Kind: 1
Desc: Count of completed GC cycles forced by the application.
Cumulative: true

### 总 GC 次数

Name: /gc/cycles/total:gc-cycles
Kind: 1
Desc: Count of all completed GC cycles.
Cumulative: true

### 触发 GC 的堆大小相对值

Name: /gc/gogc:percent
Kind: 1
Desc: Heap size target percentage configured by the user, otherwise 100. This value is set by the GOGC environment variable, and the runtime/debug.SetGCPercent function.
Cumulative: false

### 内存大小限制

Name: /gc/gomemlimit:bytes
Kind: 1
Desc: Go runtime memory limit configured by the user, otherwise math.MaxInt64. This value is set by the GOMEMLIMIT environment variable, and the runtime/debug.SetMemoryLimit function.
Cumulative: false

### 堆内存分配柱状图

Name: /gc/heap/allocs-by-size:bytes
Kind: 3
Desc: Distribution of heap allocations by approximate size. Bucket counts increase monotonically. Note that this does not include tiny objects as defined by /gc/heap/tiny/allocs:objects, only tiny blocks.
Cumulative: true

### 累计堆内存分配

Name: /gc/heap/allocs:bytes
Kind: 1
Desc: Cumulative sum of memory allocated to the heap by the application.
Cumulative: true

### 累计触发堆内存分配次数

Name: /gc/heap/allocs:objects
Kind: 1
Desc: Cumulative count of heap allocations triggered by the application. Note that this does not include tiny objects as defined by /gc/heap/tiny/allocs:objects, only tiny blocks.
Cumulative: true

### 堆内存释放柱状图

Name: /gc/heap/frees-by-size:bytes
Kind: 3
Desc: Distribution of freed heap allocations by approximate size. Bucket counts increase monotonically. Note that this does not include tiny objects as defined by /gc/heap/tiny/allocs:objects, only tiny blocks.
Cumulative: true

### 总堆内存释放

Name: /gc/heap/frees:bytes
Kind: 1
Desc: Cumulative sum of heap memory freed by the garbage collector.
Cumulative: true

### 堆内存释放次数

Name: /gc/heap/frees:objects
Kind: 1
Desc: Cumulative count of heap allocations whose storage was freed by the garbage collector. Note that this does not include tiny objects as defined by /gc/heap/tiny/allocs:objects, only tiny blocks.
Cumulative: true

### GC 循环后目标堆内存

Name: /gc/heap/goal:bytes
Kind: 1
Desc: Heap size target for the end of the GC cycle.
Cumulative: false

### 活跃堆内存

Name: /gc/heap/live:bytes
Kind: 1
Desc: Heap memory occupied by live objects that were marked by the previous GC.
Cumulative: false

### 总堆对象数

Name: /gc/heap/objects:objects
Kind: 1
Desc: Number of objects, live or unswept, occupying heap memory.
Cumulative: false

### 堆上小对象数

Name: /gc/heap/tiny/allocs:objects
Kind: 1
Desc: Count of small allocations that are packed together into blocks. These allocations are counted separately from other allocations because each individual allocation is not tracked by the runtime, only their block. Each block is already accounted for in allocs-by-size and frees-by-size.
Cumulative: true

### 上次内存限制触发时间

Name: /gc/limiter/last-enabled:gc-cycle
Kind: 1
Desc: GC cycle the last time the GC CPU limiter was enabled. This metric is useful for diagnosing the root cause of an out-of-memory error, because the limiter trades memory for CPU time when the GC's CPU time gets too high. This is most likely to occur with use of SetMemoryLimit. The first GC cycle is cycle 1, so a value of 0 indicates that it was never enabled.
Cumulative: false

### GC 暂停时间 (已废弃)

Name: /gc/pauses:seconds
Kind: 3
Desc: Deprecated. Prefer the identical /sched/pauses/total/gc:seconds.
Cumulative: true

### GC 可扫描总变量空间

Name: /gc/scan/globals:bytes
Kind: 1
Desc: The total amount of global variable space that is scannable.
Cumulative: false

### GC 可扫描所有堆空间

Name: /gc/scan/heap:bytes
Kind: 1
Desc: The total amount of heap space that is scannable.
Cumulative: false

### GC 扫描栈大小

Name: /gc/scan/stack:bytes
Kind: 1
Desc: The number of bytes of stack that were scanned last GC cycle.
Cumulative: false

### GC 扫描总空间

Name: /gc/scan/total:bytes
Kind: 1
Desc: The total amount of space that is scannable. Sum of all metrics in /gc/scan.
Cumulative: false

### 新协程栈大小

Name: /gc/stack/starting-size:bytes
Kind: 1
Desc: The stack size of new goroutines.
Cumulative: false

## 内存

### 满足释放要求的内存

Name: /memory/classes/heap/free:bytes
Kind: 1
Desc: Memory that is completely free and eligible to be returned to the underlying system, but has not been. This metric is the runtime's estimate of free address space that is backed by physical memory.
Cumulative: false

### 堆对象内存

Name: /memory/classes/heap/objects:bytes
Kind: 1
Desc: Memory occupied by live objects and dead objects that have not yet been marked free by the garbage collector.
Cumulative: false

### 已释放的内存

Name: /memory/classes/heap/released:bytes
Kind: 1
Desc: Memory that is completely free and has been returned to the underlying system. This metric is the runtime's estimate of free address space that is still mapped into the process, but is not backed by physical memory.
Cumulative: false

### 栈内存大小

Name: /memory/classes/heap/stacks:bytes
Kind: 1
Desc: Memory allocated from the heap that is reserved for stack space, whether or not it is currently in-use. Currently, this represents all stack memory for goroutines. It also includes all OS thread stacks in non-cgo programs. Note that stacks may be allocated differently in the future, and this may change.
Cumulative: false

### 未使用的堆内存

Name: /memory/classes/heap/unused:bytes
Kind: 1
Desc: Memory that is reserved for heap objects but is not currently used to hold heap objects.
Cumulative: false

### mcache 中未使用内存

Name: /memory/classes/metadata/mcache/free:bytes
Kind: 1
Desc: Memory that is reserved for runtime mcache structures, but not in-use.
Cumulative: false

### mcache 中在使用的内存

Name: /memory/classes/metadata/mcache/inuse:bytes
Kind: 1
Desc: Memory that is occupied by runtime mcache structures that are currently being used.
Cumulative: false

### mspan 中未使用的内存

Name: /memory/classes/metadata/mspan/free:bytes
Kind: 1
Desc: Memory that is reserved for runtime mspan structures, but not in-use.
Cumulative: false

### mspan 中在使用的内存

Name: /memory/classes/metadata/mspan/inuse:bytes
Kind: 1
Desc: Memory that is occupied by runtime mspan structures that are currently being used.
Cumulative: false

### 运行时保留或者 metadata 占用内存

Name: /memory/classes/metadata/other:bytes
Kind: 1
Desc: Memory that is reserved for or used to hold runtime metadata.
Cumulative: false

### 系统栈内存 cgo

Name: /memory/classes/os-stacks:bytes
Kind: 1
Desc: Stack memory allocated by the underlying operating system. In non-cgo programs this metric is currently zero. This may change in the future. In cgo programs this metric includes OS thread stacks allocated directly from the OS. Currently, this only accounts for one stack in c-shared and c-archive build modes, and other sources of stacks from the OS are not measured. This too may change in the future.
Cumulative: false

### 其他内存占用

Name: /memory/classes/other:bytes
Kind: 1
Desc: Memory used by execution trace buffers, structures for debugging the runtime, finalizer and profiler specials, and more.
Cumulative: false

### profile 内存占用

Name: /memory/classes/profiling/buckets:bytes
Kind: 1
Desc: Memory that is used by the stack trace hash map used for profiling.
Cumulative: false

### 除 cgo 外产生的所有内存

Name: /memory/classes/total:bytes
Kind: 1
Desc: All memory mapped by the Go runtime into the current process as read-write. Note that this does not include memory mapped by code called via cgo or via the syscall package. Sum of all metrics in /memory/classes.
Cumulative: false

## 调度

### 最大线程数

Name: /sched/gomaxprocs:threads
Kind: 1
Desc: The current runtime.GOMAXPROCS setting, or the number of operating system threads that can execute user-level Go code simultaneously.
Cumulative: false

### 协程数

Name: /sched/goroutines:goroutines
Kind: 1
Desc: Count of live goroutines.
Cumulative: false

### 调度延迟柱状图

Name: /sched/latencies:seconds
Kind: 3
Desc: Distribution of the time goroutines have spent in the scheduler in a runnable state before actually running. Bucket counts increase monotonically.
Cumulative: true

### GC 暂停时间柱状图

Name: /sched/pauses/stopping/gc:seconds
Kind: 3
Desc: Distribution of individual GC-related stop-the-world stopping latencies. This is the time it takes from deciding to stop the world until all Ps are stopped. This is a subset of the total GC-related stop-the-world time (/sched/pauses/total/gc:seconds). During this time, some threads may be executing. Bucket counts increase monotonically.
Cumulative: true

### 其他等待时间柱状图

Name: /sched/pauses/stopping/other:seconds
Kind: 3
Desc: Distribution of individual non-GC-related stop-the-world stopping latencies. This is the time it takes from deciding to stop the world until all Ps are stopped. This is a subset of the total non-GC-related stop-the-world time (/sched/pauses/total/other:seconds). During this time, some threads may be executing. Bucket counts increase monotonically.
Cumulative: true

### GC 暂停决策到重启的时间

Name: /sched/pauses/total/gc:seconds
Kind: 3
Desc: Distribution of individual GC-related stop-the-world pause latencies. This is the time from deciding to stop the world until the world is started again. Some of this time is spent getting all threads to stop (this is measured directly in /sched/pauses/stopping/gc:seconds), during which some threads may still be running. Bucket counts increase monotonically.
Cumulative: true

### 其他等待时间

Name: /sched/pauses/total/other:seconds
Kind: 3
Desc: Distribution of individual non-GC-related stop-the-world pause latencies. This is the time from deciding to stop the world until the world is started again. Some of this time is spent getting all threads to stop (measured directly in /sched/pauses/stopping/other:seconds). Bucket counts increase monotonically.
Cumulative: true

## 锁竞争

### 取锁累计等待时间

Name: /sync/mutex/wait/total:seconds
Kind: 2
Desc: Approximate cumulative time goroutines have spent blocked on a sync.Mutex, sync.RWMutex, or runtime-internal lock. This metric is useful for identifying global changes in lock contention. Collect a mutex or block profile using the runtime/pprof package for more detailed contention data.
Cumulative: true

## 运行时调试

GODEBUG 非默认行为指标，用于追踪运行时因 `GODEBUG` 环境变量而执行的非默认行为。所有指标 Kind 均为 `uint64`，Cumulative 为 `true`。

### os/exec

Name: /godebug/non-default-behavior/execerrdot:events
Desc: The number of non-default behaviors executed by the os/exec package due to a non-default GODEBUG=execerrdot=... setting.

### cmd/go

Name: /godebug/non-default-behavior/gocachehash:events
Desc: The number of non-default behaviors executed by the cmd/go package due to a non-default GODEBUG=gocachehash=... setting.

Name: /godebug/non-default-behavior/gocachetest:events
Desc: The number of non-default behaviors executed by the cmd/go package due to a non-default GODEBUG=gocachetest=... setting.

Name: /godebug/non-default-behavior/gocacheverify:events
Desc: The number of non-default behaviors executed by the cmd/go package due to a non-default GODEBUG=gocacheverify=... setting.

### go/types

Name: /godebug/non-default-behavior/gotypesalias:events
Desc: The number of non-default behaviors executed by the go/types package due to a non-default GODEBUG=gotypesalias=... setting.

### net/http

Name: /godebug/non-default-behavior/http2client:events
Desc: The number of non-default behaviors executed by the net/http package due to a non-default GODEBUG=http2client=... setting.

Name: /godebug/non-default-behavior/http2server:events
Desc: The number of non-default behaviors executed by the net/http package due to a non-default GODEBUG=http2server=... setting.

Name: /godebug/non-default-behavior/httplaxcontentlength:events
Desc: The number of non-default behaviors executed by the net/http package due to a non-default GODEBUG=httplaxcontentlength=... setting.

Name: /godebug/non-default-behavior/httpmuxgo121:events
Desc: The number of non-default behaviors executed by the net/http package due to a non-default GODEBUG=httpmuxgo121=... setting.

### go/build

Name: /godebug/non-default-behavior/installgoroot:events
Desc: The number of non-default behaviors executed by the go/build package due to a non-default GODEBUG=installgoroot=... setting.

### html/template

Name: /godebug/non-default-behavior/jstmpllitinterp:events
Desc: The number of non-default behaviors executed by the html/template package due to a non-default GODEBUG=jstmpllitinterp=... setting.

### mime/multipart

Name: /godebug/non-default-behavior/multipartmaxheaders:events
Desc: The number of non-default behaviors executed by the mime/multipart package due to a non-default GODEBUG=multipartmaxheaders=... setting.

Name: /godebug/non-default-behavior/multipartmaxparts:events
Desc: The number of non-default behaviors executed by the mime/multipart package due to a non-default GODEBUG=multipartmaxparts=... setting.

### net

Name: /godebug/non-default-behavior/multipathtcp:events
Desc: The number of non-default behaviors executed by the net package due to a non-default GODEBUG=multipathtcp=... setting.

### runtime

Name: /godebug/non-default-behavior/panicnil:events
Desc: The number of non-default behaviors executed by the runtime package due to a non-default GODEBUG=panicnil=... setting.

### math/rand

Name: /godebug/non-default-behavior/randautoseed:events
Desc: The number of non-default behaviors executed by the math/rand package due to a non-default GODEBUG=randautoseed=... setting.

### archive/tar

Name: /godebug/non-default-behavior/tarinsecurepath:events
Desc: The number of non-default behaviors executed by the archive/tar package due to a non-default GODEBUG=tarinsecurepath=... setting.

### crypto/tls

Name: /godebug/non-default-behavior/tls10server:events
Desc: The number of non-default behaviors executed by the crypto/tls package due to a non-default GODEBUG=tls10server=... setting.

Name: /godebug/non-default-behavior/tlsmaxrsasize:events
Desc: The number of non-default behaviors executed by the crypto/tls package due to a non-default GODEBUG=tlsmaxrsasize=... setting.

Name: /godebug/non-default-behavior/tlsrsakex:events
Desc: The number of non-default behaviors executed by the crypto/tls package due to a non-default GODEBUG=tlsrsakex=... setting.

Name: /godebug/non-default-behavior/tlsunsafeekm:events
Desc: The number of non-default behaviors executed by the crypto/tls package due to a non-default GODEBUG=tlsunsafeekm=... setting.

### crypto/x509

Name: /godebug/non-default-behavior/x509sha1:events
Desc: The number of non-default behaviors executed by the crypto/x509 package due to a non-default GODEBUG=x509sha1=... setting.

Name: /godebug/non-default-behavior/x509usefallbackroots:events
Desc: The number of non-default behaviors executed by the crypto/x509 package due to a non-default GODEBUG=x509usefallbackroots=... setting.

Name: /godebug/non-default-behavior/x509usepolicies:events
Desc: The number of non-default behaviors executed by the crypto/x509 package due to a non-default GODEBUG=x509usepolicies=... setting.

### archive/zip

Name: /godebug/non-default-behavior/zipinsecurepath:events
Desc: The number of non-default behaviors executed by the archive/zip package due to a non-default GODEBUG=zipinsecurepath=... setting.

# 接入监控系统

## Prometheus

使用 `prometheus/client_golang` 将 runtime/metrics 桥接到 Prometheus：

```go
package main

import (
	"runtime/metrics"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"net/http"
)

func main() {
	// 注册一个 Gauge 指标，由 collect 函数动态更新
	prometheus.MustRegister(prometheus.NewGaugeFunc(
		prometheus.GaugeOpts{
			Name: "go_heap_free_bytes",
			Help: "Memory eligible to be returned to the system.",
		},
		func() float64 {
			samples := []metrics.Sample{{Name: "/memory/classes/heap/free:bytes"}}
			metrics.Read(samples)
			return float64(samples[0].Value.Uint64())
		},
	))

	http.Handle("/metrics", promhttp.Handler())
	http.ListenAndServe(":2112", nil)
}
```

## OpenTelemetry

通过 OTel 的 `runtime` metric 包自动采集：

```go
import (
	"go.opentelemetry.io/otel/exporters/prometheus"
	"go.opentelemetry.io/otel/sdk/metric"
)

exporter, _ := prometheus.New()
provider := metric.NewMeterProvider(metric.WithReader(exporter))
// Go 运行时指标会自动注册到 provider
```

详细配置参考各 SDK 文档，核心思路一致：定期调用 `metrics.Read`，将值写入监控系统的数据模型。

# 常见问题

## metrics.Read 是并发安全的吗？

是的。`metrics.Read` 可以从多个 goroutine 同时调用，内部有适当的同步机制。在高并发服务中，各 goroutine 可以独立采样自己关心的指标。

## metrics.All() 返回的切片可以缓存吗？

可以。`metrics.All()` 返回的 `Description` 切片在进程生命周期内不会变化，适合在初始化时缓存一次，后续复用。例如构建仪表盘或动态选择采样目标时：

```go
var allMetrics = metrics.All() // 启动时缓存
```

## Cumulative 值在进程重启后会怎样？

会归零。Cumulative 是进程级别的累计值，重启后重新开始计数。如果你用 Cumulative 值做差计算速率（如分配速率），需要确保两次采样在同一个进程生命周期内。跨重启的场景应使用瞬时值（Cumulative: false）的指标。

## 为什么读到 KindBad？

表示该指标在当前 Go 版本中已被移除或重命名。处理方式：

```go
metrics.Read(samples)
if samples[0].Value.Kind() == metrics.KindBad {
	// 指标不存在，跳过或降级处理
	return
}
```

建议在代码中始终做 Kind 检查，避免因 Go 版本升级导致 panic。

## Histogram 的 Bucket 是固定的吗？

是的。`Float64Histogram.Buckets` 在进程生命周期内不会变，可以缓存。但不同 Go 版本的 Bucket 边界可能不同，升级 Go 后应重新获取。

# 参考

[metrics 文档](https://pkg.go.dev/runtime/metrics)
