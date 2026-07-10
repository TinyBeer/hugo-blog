---
date: '2024-06-07T22:57:50+08:00'
title: 'slice 原理&实践'
tags: ['Golang', 'slice']
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

slice 是 golang 中最常使用的基础数据结构之一。区别于数组，slice 是一个引用类型。<!-- more -->

# 基础使用

## 声明

```go
// int 可以替换为任意类型标识
var slice1 []int               // nil切片
slice2 := []int{}              // 空切片 len 0 cap 0
slice3 := []int{1,2,3}         // len 3 cap 3
slice4 := make([]int, 1)       // len 1 cap 1
slice5 := make([]int, 1, 2)    // len 1 cap 2
```

### nil 切片与空切片

```go
var s1 []int        // nil 切片
s2 := []int{}       // 空切片
s3 := make([]int, 0) // 空切片
```

三者 `len` 和 `cap` 都是 0，行为几乎一致，但有细微差别：

| 对比项 | nil 切片 | 空切片 |
|--------|----------|--------|
| 底层数组指针 | nil | 指向 runtime.zerobase |
| `== nil` 判断 | true | false |
| `json.Marshal` | `null` | `[]` |
| `append` | 正常工作 | 正常工作 |

实际开发中，函数返回值建议统一用 `return nil` 或 `return s`，不需要刻意返回空切片。在 API 场景下，如果需要 JSON 输出 `[]` 而非 `null`，则初始化为空切片。

## 访问数据

```go
item1 := slice1[0]      // 下标取值  下标从0开始
for i := range slice    // 遍历下标
for i, v := range slice // 遍历下标和值
```

## 操作

### 拼接

```go
newSlice := append(slice, item1, item2, ...)    // 追加元素
newSlice := append(slice1, slice2...)           // 追加切片
```

`slices` 包还提供了更简洁的拼接方式，详见下方 `slices 包常用函数` 章节。

### slices 包常用函数

go1.21 引入 `slices` 泛型包，提供类型安全的切片操作，比手写循环更简洁：

```go
import "slices"

s := []int{3, 1, 4, 1, 5}

// 排序
slices.Sort(s)                        // [1, 1, 3, 4, 5]
slices.SortFunc(s, func(a, b int) int { return b - a })  // 自定义排序

// 查找
slices.Contains(s, 3)                 // true
slices.Index(s, 4)                    // 3
slices.BinarySearch(s, 3)             // 2（需已排序）

// 修改
slices.Delete(s, 1, 3)               // 删除下标 [1,3) 的元素
slices.Compact(s)                     // 去重（需已排序）
slices.Reverse(s)                     // 原地翻转

// 拼接
newSlice := slices.Concat(s1, s2, s3) // go1.22+ 多切片拼接
```

相比传统写法，`slices` 包避免了下标越界风险，且编译器可针对泛型做内联优化。

### 裁剪

裁剪操作的本质是创建新的 slice 头，公用存储空间，并不会申请新的存储空间。没有发生扩容的情况下，对裁剪下来的切片进行数据修改会影响原切片。这里的操作结果是不可预测的，使用时应当充分小心。

```go
slice1[start:end]   // 裁剪从start开始，到end结束不包含end下标的元素  裁剪后的cap为原cap-start
slice2[:end]        // 省略start表示从0开始
slice3[start:]      // 省略end表示在最后一个元素处结束
slice4[:]           // 裁剪出所有元素
```

用例：移除切片中的元素

```go
slice1 = slice1[1:]                        // 移除首个元素
slice2 = slice2[:len(slice2)-1]            // 移除尾部元素
slice = append(slice[:i], slice[i+1:])     // 移除下标为i的元素  需要注意下标是否超限
```

### 拷贝

```go
newSlice := slice  // 浅拷贝，仅拷贝头部信息
// copy 的原则是在不改变长度的情况下尽可能进行拷贝
// 示例：dst=[]int{9, 9} src=[]int{1, 2, 3, 4}
// 结果：dst=[]int{1, 2} src=[]int{1, 2, 3, 4}
copy(dst, src)  // 尽可能将src中的元素拷贝到dst中  保持下标一致
```

### 深拷贝

浅拷贝只复制 slice header，底层数组仍共享。深拷贝需要分配新的底层数组：

```go
// 方法一：copy + make
src := []int{1, 2, 3}
dst := make([]int, len(src))
copy(dst, src)
dst[0] = 99  // 不影响 src

// 方法二：append
dst := append([]int(nil), src...)

// 方法三：go1.21+ slices.Clone
dst := slices.Clone(src)
```

三种方法本质相同：分配新数组 + 逐元素复制。

## 传参

```go
// 函数签名  func test([]int)
test(slice)  // 引用传递，不发生扩容的情况下修改切片中的值会影响原切片
```

如果想要使用函数处理后的 slice，可以有以下几个方法：

- 将 slice 作为参数返回（推荐）
  - slice 作为引用类型返回只会拷贝头部信息，数据量并不大。
- 使用 slice 指针进行传参
  - 如果使用指针传递会发生内存逃逸，虽然扩容也可能触发逃逸，但在大部分情况下作为参数传递是更高效的。
- 保证 slice 的容量足够，不会在函数中发生扩容
  - 大多数情况下我们无法正确估计 slice 的容量，给出冗余量又会造成浪费，极端情况下还会发生逃逸。

## panic

slice 在以下两种情况下会触发 panic

- 下标超出限制
  - 使用下标访问 nil 切片
  - 访问下标超过 len 所限定范围
- make 时 cap 过大，实际编码很少遇到
  - 32 位系统设置的值超过 int32 的最大值
  - 所需分配空间超过计算机寻址空间

## 常见陷阱

### range 遍历时修改 slice

在 `for range` 中修改 value 不会影响原 slice，因为 value 是副本：

```go
s := []int{1, 2, 3}
for _, v := range s {
    v = 0  // 不会影响 s
}
// s 仍为 []int{1, 2, 3}
```

但如果需要修改原 slice，应通过下标操作：

```go
s := []int{1, 2, 3}
for i := range s {
    s[i] = 0
}
// s 变为 []int{0, 0, 0}
```

### 切片扩容后的悬挂引用

切片扩容后底层数组可能重新分配，旧切片的引用不会自动更新：

```go
s := make([]int, 0, 2)
a := s[:0]
a = append(a, 1, 2, 3)  // 触发扩容，a 的底层数组已变
fmt.Println(s)  // []  s 仍指向旧数组
```

### 切片作为 map value 的并发问题

并发 **读取** map 中的 slice value 是安全的，但以下场景会导致 data race：

- 并发对同一个 slice 变量做 `append`（slice header 非原子操作）
- 并发修改 map 本身（增删 key）

应加锁保护或使用 channel 传递。

### nil 切片和空切片的 JSON 序列化

两者 `len` 和 `cap` 都是 0，但 JSON 序列化结果不同：

```go
var a []int       // nil 切片
b := []int{}      // 空切片
json.Marshal(a)   // null
json.Marshal(b)   // []
```

在 API 返回中应明确使用空切片以避免前端收到 `null`。

## 性能优化

### 预分配容量

已知元素数量时用 `make` 预分配，避免频繁扩容：

```go
// 不推荐
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)  // 多次扩容
}

// 推荐
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)  // 零扩容
}
```

### 传参优先返回而非指针

返回 slice 只拷贝 header（24 字节），传指针会导致内存逃逸到堆上：

```go
// 不推荐
func fill(s *[]int) {
    *s = append(*s, 1, 2, 3)
}

// 推荐
func fill() []int {
    s := make([]int, 0, 3)
    return append(s, 1, 2, 3)
}
```

### 复用底层数组

通过 `slice[:0]` 重置 len 可复用已有内存，避免重新分配：

```go
buf := make([]byte, 0, 1024)
for {
    buf = buf[:0]  // 复用底层数组
    buf = append(buf, data...)
    process(buf)
}
```

# 源码分析

> 源码版本 go1.22
> 源码路径：`src/runtime/slice.go`

## slice 底层结构

```go
type slice struct {
	array unsafe.Pointer   // 指向数据存储地址（一片连续的内存空间）的指针
	len   int              // 保存切片长度
	cap   int              // 保存切片容量  cap*sizeof(element)就是存储空间大小
}
// A notInHeapSlice is a slice backed by runtime/internal/sys.NotInHeap memory.
type notInHeapSlice struct {
	array *notInHeap
	len   int
	cap   int
}
```

## make 逻辑

- makeslice
  1. 参数校验
  2. mallocgc 申请内存资源
     - mem：申请的空间大小
     - et：元素类型
     - true：初始化为零值

```go
//go:linkname makeslice
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

## 扩容逻辑

- growslice
  - oldPtr: 老切片的 array 所指向的地址
  - newLen: 新的切片长度 oldLen + num
  - oldCap: 老切片的容量
  - num: 要添加的元素长度
  - et: 元素类型

1. 参数校验，处理参数异常，过滤掉不需要新空间的情况
2. 计算所需要的新空间大小
   - nextslicecap 确定新的 cap 大小
   - 空间计算使用了三种方法优化，主要是规避除法计算，使用移位替代乘法
3. 申请新的内存空间
   - 如果元素类型不包含指针，清理尾部不会写数据的 cap-len 长度的空间
   - 如果包含指针，处理和 gc 相关的指针映射
4. 拷贝老切片的数据到新的切片

```go
//go:linkname growslice
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
	oldLen := newLen - num
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(oldPtr, uintptr(oldLen*int(et.Size_)), callerpc, abi.FuncPCABIInternal(growslice))
	}
	if msanenabled {
		msanread(oldPtr, uintptr(oldLen*int(et.Size_)))
	}
	if asanenabled {
		asanread(oldPtr, uintptr(oldLen*int(et.Size_)))
	}

	if newLen < 0 {
		panic(errorString("growslice: len out of range"))
	}

	if et.Size_ == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve oldPtr in this case.
		return slice{unsafe.Pointer(&zerobase), newLen, newLen}
	}

	newcap := nextslicecap(newLen, oldCap)

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.Size.
	// For 1 we don't need any division/multiplication.
	// For goarch.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	noscan := !et.Pointers()
	switch {
	case et.Size_ == 1:
		lenmem = uintptr(oldLen)
		newlenmem = uintptr(newLen)
		capmem = roundupsize(uintptr(newcap), noscan)
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.Size_ == goarch.PtrSize:
		lenmem = uintptr(oldLen) * goarch.PtrSize
		newlenmem = uintptr(newLen) * goarch.PtrSize
		capmem = roundupsize(uintptr(newcap)*goarch.PtrSize, noscan)
		overflow = uintptr(newcap) > maxAlloc/goarch.PtrSize
		newcap = int(capmem / goarch.PtrSize)
	case isPowerOfTwo(et.Size_):
		var shift uintptr
		if goarch.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.TrailingZeros64(uint64(et.Size_))) & 63
		} else {
			shift = uintptr(sys.TrailingZeros32(uint32(et.Size_))) & 31
		}
		lenmem = uintptr(oldLen) << shift
		newlenmem = uintptr(newLen) << shift
		capmem = roundupsize(uintptr(newcap)<<shift, noscan)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
		capmem = uintptr(newcap) << shift
	default:
		lenmem = uintptr(oldLen) * et.Size_
		newlenmem = uintptr(newLen) * et.Size_
		capmem, overflow = math.MulUintptr(et.Size_, uintptr(newcap))
		capmem = roundupsize(capmem, noscan)
		newcap = int(capmem / et.Size_)
		capmem = uintptr(newcap) * et.Size_
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: len out of range"))
	}

	var p unsafe.Pointer
	if !et.Pointers() {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from oldLen to newLen.
		// Only clear the part that will not be overwritten.
		// The reflect_growslice() that calls growslice will manually clear
		// the region not cleared here.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in oldPtr since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			//
			// It's safe to pass a type to this function as an optimization because
			// from and to only ever refer to memory representing whole values of
			// type et. See the comment on bulkBarrierPreWrite.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(oldPtr), lenmem-et.Size_+et.PtrBytes, et)
		}
	}
	memmove(p, oldPtr, lenmem)

	return slice{p, newLen, newcap}
}
```

- nextslicecap 返回新切片的容量，计算逻辑逐级匹配：

1. 如果新切片超过老切片容量的两倍则使用新切片长度作为容量
2. 如果老切片容量小于阈值 256 返回两倍老切片容量
3. 使用公式`newcap += (newcap + 3*threshold) >> 2` 计算直到 newcap 大于 newLen, 这是翻倍扩容和 1.25 倍扩容间的平滑处理
4. 如果发生移除则返回 newLen

```go
// nextslicecap computes the next appropriate slice length.
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		// Transition from growing 2x for small slices
		// to growing 1.25x for large slices. This formula
		// gives a smooth-ish transition between the two.
		newcap += (newcap + 3*threshold) >> 2

		// We need to check `newcap >= newLen` and whether `newcap` overflowed.
		// newLen is guaranteed to be larger than zero, hence
		// when newcap overflows then `uint(newcap) > uint(newLen)`.
		// This allows to check for both with the same comparison.
		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	// Set newcap to the requested cap when
	// the newcap calculation overflowed.
	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

# 总结

| 维度 | 要点 |
|------|------|
| 底层结构 | `array` 指针 + `len` + `cap`，三字段构成 slice header（24 字节） |
| 核心特性 | 引用类型，赋值/传参仅拷贝 header，底层数组共享 |
| 扩容规则 | < 256 翻倍，>= 256 约 1.25 倍增长，扩容可能重新分配底层数组 |
| 性能关键 | 预分配 cap、返回 slice 优于传指针、复用 `[:0]` 减少分配 |
| 安全注意 | 下标越界 panic、range value 是副本、并发 append 不安全 |

掌握 slice 的内存模型和扩容机制，是写出高性能、无 bug Go 代码的基础。
