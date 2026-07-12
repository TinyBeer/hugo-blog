---
date: "2024-05-29T19:34:19+08:00"
title: "Golang  -- sync.Pool 从使用到源码"
tags: ["Golang", "sync.Pool"]
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

`sync.Pool`是一个协程安全的临时对象存储池。主要用于缓存频繁使用的对象，以减少 GC 压力。<!--more-->

# 使用

先来看官方提供的示例，利用`sync.Pool`实现的日志打印函数：

```go
var bufPool = sync.Pool{
	New: func() any {
		// The Pool's New function should generally only return pointer
		// types, since a pointer can be put into the return interface
		// value without an allocation:
		return new(bytes.Buffer)
	},
}

// timeNow is a fake version of time.Now for tests.
func timeNow() time.Time {
	return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
	b := bufPool.Get().(*bytes.Buffer)
	b.Reset()
	// Replace this with time.Now() in a real logger.
	b.WriteString(timeNow().UTC().Format(time.RFC3339))
	b.WriteByte(' ')
	b.WriteString(key)
	b.WriteByte('=')
	b.WriteString(val)
	w.Write(b.Bytes())
	bufPool.Put(b)
}

func main() {
	Log(os.Stdout, "path", "/search?q=flowers")
}
```

核心模式如下：

1. 创建`Pool`对象并指定`New`方法（用于创建缓存对象）。
2. `Get`取用对象，使用前通过`Reset`清洗上一次的使用痕迹。
3. 使用对象完成业务逻辑。
4. `Put`归还对象。
5. 重复步骤 2-4。

## 常见错误

**必须 Put 归还**

Get 到的对象用完后必须 Put 归还，否则对象不会被复用，Pool 退化为每次创建新对象：

```go
// 错误：忘记 Put，内存泄漏且失去复用意义
b := bufPool.Get().(*bytes.Buffer)
b.WriteString(data)
w.Write(b.Bytes())
// 缺少 bufPool.Put(b)
```

**使用前必须 Reset**

从 Pool 中取出的对象可能残留上一次的数据，使用前必须重置：

```go
b := bufPool.Get().(*bytes.Buffer)
// 错误：没有 Reset，上次的数据还在
b.WriteString(newData)

// 正确：
b.Reset()
b.WriteString(newData)
```

**不要缓存带状态的对象（除非明确重置）**

如果对象有内部状态（如 bytes.Buffer 的内容、切片的长度），取出后必须重置到初始状态再使用。

## 不适用场景

1. 需要长期持有的对象（如数据库连接），因为 Pool 中的对象不保证存活，可能被 GC 回收。
2. 保存数据，无法预测 GC 何时回收其中的对象。
3. 创建代价高昂的对象，Pool 的复用收益可能不足以覆盖其复杂性。

# 应用场景

## 临时 slice 复用

避免频繁分配和释放切片：

```go
var slicePool = sync.Pool{
	New: func() any {
		s := make([]byte, 0, 1024)
		return &s
	},
}

func process(data []byte) {
	buf := slicePool.Get().(*[]byte)
	defer slicePool.Put(buf)
	*buf = (*buf)[:0]
	// 使用 *buf 进行处理
}
```

## JSON 编解码器复用

`json.Encoder` 和 `json.Decoder` 的创建开销较大，可以复用：

```go
var encoderPool = sync.Pool{
	New: func() any {
		return json.NewEncoder(nil)
	},
}

// 注意：需要通过 Reset 重新设置 writer
```

> **提示**：标准库内部已经大量使用 sync.Pool，如 `fmt` 包的打印缓冲区、`encoding/json` 的编解码器等。

# 源码解析

## 数据结构

![struct.png](https://s2.loli.net/2024/05/29/QRx2TqmSzKbD7v1.png)

### Pool

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() any
}
```

### noCopy

`noCopy`是一个空结构体，实现了`Lock()`和`Unlock()`方法。将其嵌入结构体后，`go vet`会在检测到该结构体被赋值拷贝时发出警告。正常编译不受影响。

```go
var bufPool = sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func main() {
	_ = bufPool
}
```

```sh
go vet .
# test
# [test]
./main.go:15:6: assignment copies lock value to _: sync.Pool contains sync.noCopy
```

### local 与 localSize

一个固定长度的本地对象池数组，类型为`[P]poolLocal`，其中 P 对应`runtime.GOMAXPROCS(0)`的返回值（即逻辑处理器数量）。每次 GC 后，旧 local 会被丢弃并重新分配。localSize 记录 local 的实际长度，用于遍历时确定边界。

### victim 与 victimSize

victim 和 victimSize 是上一轮的 local 和 localSize。每次 GC 时，原 victim 被回收，当前 local 移入 victim。这样对象至少能存活一个 GC 周期，避免高负载下 GC 后对象全部丢失导致的性能抖动。

### New

用户自定义的创建对象的函数。

### poolLocal

```go
type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
	private any       // Can be used only by the respective P.
	shared  poolChain // Local P can pushHead/popHead; any P can popTail.
}

type poolChain struct {
	// head is the poolDequeue to push to. This is only accessed
	// by the producer, so doesn't need to be synchronized.
	head *poolChainElt

	// tail is the poolDequeue to popTail from. This is accessed
	// by consumers, so reads and writes must be atomic.
	tail *poolChainElt
}

type poolChainElt struct {
	poolDequeue

	// next and prev link to the adjacent poolChainElts in this
	// poolChain.
	//
	// next is written atomically by the producer and read
	// atomically by the consumer. It only transitions from nil to
	// non-nil.
	//
	// prev is written atomically by the consumer and read
	// atomically by the producer. It only transitions from
	// non-nil to nil.
	next, prev *poolChainElt
}

type poolDequeue struct {
	// headTail packs together a 32-bit head index and a 32-bit
	// tail index. Both are indexes into vals modulo len(vals)-1.
	//
	// tail = index of oldest data in queue
	// head = index of next slot to fill
	//
	// Slots in the range [tail, head) are owned by consumers.
	// A consumer continues to own a slot outside this range until
	// it nils the slot, at which point ownership passes to the
	// producer.
	//
	// The head index is stored in the most-significant bits so
	// that we can atomically add to it and the overflow is
	// harmless.
	headTail atomic.Uint64

	// vals is a ring buffer of interface{} values stored in this
	// dequeue. The size of this must be a power of 2.
	//
	// vals[i].typ is nil if the slot is empty and non-nil
	// otherwise. A slot is still in use until *both* the tail
	// index has moved beyond it and typ has been set to nil. This
	// is set to nil atomically by the consumer and read
	// atomically by the producer.
	vals []eface
}

type eface struct {
	typ, val unsafe.Pointer
}
```

- poolLocalInternal 是数据的存储空间，其中 private 是当前 P 私有的空间，可以存取一个对象。
- shared 是一个双向链表。头结点只有当前 P 能访问（只有当前 P 能放入对象），所以不用做并发控制；尾结点所有 P 都能访问，需要做并发控制。shared 中的每个结点 poolChainElt 保存的是一个用数组实现的循环链表 poolDequeue。
- poolDequeue 中 headTail 为一个 uint64 数据，高 32 位和低 32 位分别保存循环链表的头尾索引。数据元素为 eface。循环队列满时会创建一个两倍长度的新队列。
- eface 是两个指针，分别保存对象的类型和值。

### Pin 机制

Get 和 Put 源码中都出现了 `pin()` 和 `runtime_procUnpin()` 调用（详见下方[关键流程](#关键流程)）。Pin 的作用是将当前 goroutine 绑定到当前逻辑处理器 P 上，防止在 Pool 操作期间被调度器抢占到其他 P。

如果没有 pin，goroutine 可能在 Get 获取 private 对象后、读取 shared 之前被抢占到另一个 P，导致访问错误的 poolLocal。Pin 通过临时禁用抢占式调度，保证整个 Get/Put 操作在同一个 P 上原子完成。

## 关键流程

### Get

```go
func (p *Pool) Get() any {
	if race.Enabled {
		race.Disable()
	}
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
		x, _ = l.shared.popHead()
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

**Get 流程**

1. 调用 pin 获取当前 P 的 poolLocal，从 private 中获取对象，若有则直接返回。
2. 否则从 shared 的头部获取。
3. 否则执行 getSlow（从其他 P 偷取或从 victim 中获取）。
4. 若仍无可用对象，调用`New`创建新对象。

```go
func (p *Pool) getSlow(pid int) any {
	// See the comment in pin regarding ordering of the loads.
	size := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	locals := p.local                            // load-consume
	// Try to steal one element from other procs.
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Try the victim cache. We do this after attempting to steal
	// from all primary caches because we want objects in the
	// victim cache to age out if at all possible.
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Mark the victim cache as empty for future gets don't bother
	// with it.
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

**getSlow 流程**

1. 从其他 P 的 shared 尾部偷取对象。
2. 从 victim 中按 private → shared 的顺序获取。
3. 标记 victim 为空后返回 nil。

### Put

```go
// Put adds x to the pool.
func (p *Pool) Put(x any) {
	if x == nil {
		return
	}
	if race.Enabled {
		if runtime_randn(4) == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	l, _ := p.pin()
	if l.private == nil {
		l.private = x
	} else {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}
```

**Put 流程**

1. 优先存入当前 P 的 private。
2. private 已有值时存入 shared 头部。

```go
// pushHead 向 shared 链表头部插入对象，队列满时双倍扩容
func (c *poolChain) pushHead(val any) {
	d := c.head
	if d == nil {
		// Initialize the chain.
		const initSize = 8 // Must be a power of 2
		d = new(poolChainElt)
		d.vals = make([]eface, initSize)
		c.head = d
		storePoolChainElt(&c.tail, d)
	}

	if d.pushHead(val) {
		return
	}

	// The current dequeue is full. Allocate a new one of twice
	// the size.
	newSize := len(d.vals) * 2
	if newSize >= dequeueLimit {
		// Can't make it any bigger.
		newSize = dequeueLimit
	}

	d2 := &poolChainElt{prev: d}
	d2.vals = make([]eface, newSize)
	c.head = d2
	storePoolChainElt(&d.next, d2)
	d2.pushHead(val)
}
```

### Race Detection

源码中多处 `race.Enabled` 分支是 Go 竞态检测器的逻辑。当启用 `-race` 编译时：

- **Get 中**：获取对象后调用 `race.Acquire`，标记该对象被当前 goroutine 占有。
- **Put 中**：以 1/4 的概率随机丢弃对象（`runtime_randn(4) == 0`），调用 `race.ReleaseMerge` 释放竞态追踪。这样做的目的是模拟对象被 GC 回收的场景，帮助检测代码是否正确处理了 Pool 对象丢失的情况。

这是 Pool 的防御性设计——你的代码必须能容忍 Get 返回 nil 或对象被意外丢弃。

## GC 与版本演进

### GC 机制

`sync.Pool`会在每次 GC 时清理池中对象。Go 1.13 引入了 victim cache 机制（详见上方 victim 与 victimSize 一节），对象至少能存活一个 GC 周期，避免高负载下 GC 后对象全部丢失导致的性能抖动。

### 版本演进

| 版本 | 变化                                    |
| ---- | --------------------------------------- |
| 1.3  | 引入 sync.Pool，每次 GC 完全清空        |
| 1.13 | 引入 victim cache，对象存活一个 GC 周期 |
| 1.21 | 优化 GC 清理逻辑，减少 STW 开销         |
