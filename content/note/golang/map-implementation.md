---
date: "2021-04-18T10:03:19+08:00"
title: "Map -- 底层实现"
tags: ["Golang", "源码阅读"]
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

---

title: Go Map 底层实现
date: 2021-04-18
tags: ['Golang', '数据结构']
toc: true

---

# Go Map 底层实现

## 概述

Go 语言的 map 实现基于**哈希查找表（Hash Table）**，使用**链地址法**解决哈希冲突。相比自平衡搜索树（AVL、红黑树）的 O(logN) 最差效率，哈希表平均 O(1) 的查找性能更适合作为通用容器。

本文基于 Go 1.16 版本分析。

### 核心特点

- 平均 O(1) 的增删查改
- 非并发安全（读写冲突会 panic）
- 遍历顺序随机
- 底层使用链表法解决冲突，单桶容量为 8

## 存储结构

### hmap 结构

hmap 是 map 的顶层结构，通过 `runtime.mapassign` 等函数操作：

```go
type hmap struct {
    count     int            // 元素数量，len() 返回值
    flags     uint8          // 状态标志（见下表）
    B         uint8          // 桶数量 = 2^B
    noverflow uint16         // 溢出桶近似数量
    hash0     uint32         // 哈希种子，用于 hash(key, hash0)
    buckets   unsafe.Pointer // 桶数组指针
    oldbuckets unsafe.Pointer // 扩容时的旧桶数组，非 nil 时表示正在扩容
    nevacuate uintptr        // 扩容迁移进度，已迁移的桶数量
    extra     *mapextra      // 可选，溢出桶存储
}
```

**flags 标志位：**

| 位  | 名称         | 含义               |
| :-- | :----------- | :----------------- |
| 0   | hashWriting  | 有协程正在写       |
| 1   | iterator     | 有迭代器在使用桶   |
| 2   | oldIterator  | 有迭代器在使用旧桶 |
| 3   | sameSizeGrow | 正在进行等量扩容   |

### bmap 桶结构

每个桶（bucket）可以存放 **8 个键值对**：

```go
// 表面定义
type bmap struct {
    tophash [bucketCnt]uint8  // bucketCnt = 8
}

// 编译器实际生成的结构
type bmap struct {
    topbits  [8]uint8     // 每个位置的 top hash（高8位）
    keys     [8]keytype   // 8个key
    values   [8]valuetype // 8个value
    pad      uintptr
    overflow uintptr      // 溢出桶指针
}
```

**tophash 的作用：**

- 保存 hash 值的高 8 位，用于快速定位和比较
- 遍历时先比较 tophash，不匹配直接跳过，避免频繁比较 key

### mapextra 结构

当 key/value 不含指针时，overflow 指针会移到 mapextra，避免 GC 扫描：

```go
type mapextra struct {
    overflow    *[]*bmap  // 当前使用的溢出桶
    oldoverflow *[]*bmap  // 扩容时的旧溢出桶
    nextOverflow *bmap    // 预分配的下一个溢出桶
}
```

## 哈希映射原理

### 桶定位

```
hash = hasher(key, hash0)    // 计算 64 位哈希值
bucket = hash & (2^B - 1)    // 低 B 位定位桶
tophash = hash >> 56         // 高 8 位用于桶内快速匹配
```

**示例：** B=5 时，桶数量 = 32

- hash 低 5 位 = `01100` = 12，定位到第 12 个桶
- hash 高 8 位用于在桶内 8 个位置中快速定位

### 桶内查找

1. 用 tophash 快速筛选候选位置
2. 匹配到后，比较完整的 key
3. 如果当前桶未找到，沿 overflow 链继续查找

## 基本操作

### 创建 map

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // hint 是预估的元素数量
    // 计算合适的 B 值
    B := uint8(0)
    for overLoadFactor(hint, B) {
        B++
    }
    h.B = B

    // 分配桶数组，多余的作为溢出桶预分配
    if h.B != 0 {
        h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
    }
    return h
}
```

### 查询 mapaccess

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0])  // 返回零值
    }

    // 并发检测
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }

    hash := t.hasher(key, uintptr(h.hash0))
    m := bucketMask(h.B)
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))

    // 如果正在扩容，优先从旧桶查找
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            m >>= 1
        }
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        if !evacuated(oldb) {
            b = oldb
        }
    }

    top := tophash(hash)
bucketloop:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    break bucketloop  // 后面没有元素了
                }
                continue
            }
            // 定位 key
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
            if t.key.equal(key, k) {
                // 定位 value
                e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                return e
            }
        }
    }
    return unsafe.Pointer(&zeroVal[0])
}
```

### 插入/更新 mapassign

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
    hash := t.hasher(key, uintptr(h.hash0))
    h.flags ^= hashWriting  // 设置写标志

again:
    bucket := hash & bucketMask(h.B)
    if h.growing() {
        growWork(t, h, bucket)  // 增量迁移
    }
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
    top := tophash(hash)

    var inserti *uint8     // 待插入的 tophash 位置
    var insertk unsafe.Pointer  // 待插入的 key 位置
    var elem unsafe.Pointer     // 待插入的 value 位置

bucketloop:
    for {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                if isEmpty(b.tophash[i]) && inserti == nil {
                    inserti = &b.tophash[i]
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
                    elem = add(insertk, bucketCnt*uintptr(t.keysize))
                }
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            // key 已存在，更新 value
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            if t.indirectkey() {
                k = *((*unsafe.Pointer)(k))
            }
            if t.key.equal(key, k) {
                elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
                goto done
            }
        }
        ovf := b.overflow(t)
        if ovf == nil {
            break
        }
        b = ovf
    }

    // 检查是否需要扩容
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)  // 扩容（不迁移）
        goto again
    }

    // 找到空位插入
    if inserti == nil {
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        elem = add(insertk, bucketCnt*uintptr(t.keysize))
    }

    typedmemmove(t.key, insertk, key)
    *inserti = top
    h.count++

done:
    h.flags &^= hashWriting  // 清除写标志
    return elem
}
```

### 删除 mapdelete

```go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
    hash := t.hasher(key, uintptr(h.hash0))
    h.flags ^= hashWriting

    bucket := hash & bucketMask(h.B)
    if h.growing() {
        growWork(t, h, bucket)
    }
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
    top := tophash(hash)

search:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < bucketCnt; i++ {
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    break search
                }
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            k2 := k
            if t.indirectkey() {
                k2 = *((*unsafe.Pointer)(k2))
            }
            if !t.key.equal(key, k2) {
                continue
            }
            // 清理 key/value
            if t.indirectkey() {
                *(*unsafe.Pointer)(k) = nil
            } else if t.key.ptrdata != 0 {
                memclrHasPointers(k, t.key.size)
            }
            e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
            if t.indirectelem() {
                *(*unsafe.Pointer)(e) = nil
            } else if t.elem.ptrdata != 0 {
                memclrHasPointers(e, t.elem.size)
            } else {
                memclrNoHeapPointers(e, t.elem.size)
            }
            b.tophash[i] = emptyOne
            h.count--
            break search
        }
    }
    h.flags &^= hashWriting
}
```

## 扩容机制

### 触发条件

| 扩容类型 | 触发条件                  | 结果       |
| :------- | :------------------------ | :--------- |
| 翻倍扩容 | 装填因子 > 6.5            | 桶数量翻倍 |
| 等量扩容 | 溢出桶数量 > 2^min(B, 15) | 桶数量不变 |

**装填因子 = count / (2^B)**

### 增量迁移

扩容采用**渐进式迁移**，在每次插入/删除操作时迁移 1-2 个桶：

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    // 迁移当前操作的桶
    b := (*bmap)(add(h.oldbuckets, bucket*uintptr(t.bucketsize)))
    evacuate(t, h, b)
}
```

### 迁移策略

evacuate 函数将旧桶数据按两种情况迁移：

- **evacuatedX**：键迁移位置 < 旧桶长度
- **evacuatedY**：键迁移位置 ≥ 旧桶长度（翻倍扩容时使用）

## 遍历机制

### 随机起点

Go map 遍历从**随机桶**开始，遍历顺序完全随机：

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
    r := uintptr(fastrand())
    it.startBucket = r & bucketMask(h.B)
    it.offset = uint8(r >> h.B & (bucketCnt - 1))
    // ...
}
```

**设计原因：** 避免开发者依赖遍历顺序。

### 迭代器结构

```go
type hiter struct {
    key           unsafe.Pointer
    elem          unsafe.Pointer
    t             *maptype
    h             *hmap
    buckets       unsafe.Pointer
    bptr          *_bmap       // 当前桶
    overflow      []*_bmap     // 溢出桶引用
    oldoverflow   []*bmap      // 旧溢出桶引用
    startBucket   uintptr      // 起始桶
    offset        uint8        // 桶内起始偏移
    wrapped       bool         // 是否回到起点
    B             uint8
    i             uint8        // 桶内当前位置
    bucket        uintptr      // 当前桶号
    checkBucket   uintptr      // 扩容时检查的桶
}
```

## 并发安全

Go map **不是并发安全的**，读写/写写冲突会直接 panic：

```go
// 读操作检测
if h.flags&hashWriting != 0 {
    throw("concurrent map read and map write")
}

// 写操作检测
if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
}
```

**并发场景解决方案：**

| 方案         | 适用场景           | 特点       |
| :----------- | :----------------- | :--------- |
| sync.Mutex   | 写多读少           | 简单可靠   |
| sync.RWMutex | 读多写少           | 提升读性能 |
| sync.Map     | 读多写少、key 稳定 | 官方实现   |
| 分片 map     | 高并发写           | 减少锁竞争 |

## 使用要点

### 类型约束

```go
// key 必须是 comparable
m1 := map[int]string{}      // OK
m2 := map[interface{}]int{} // OK
m3 := map[[]int]int{}       // 编译错误：slice 不可比较
```

### 零值处理

```go
var m map[string]int
// m == nil，读取返回零值，写入 panic

m = make(map[string]int)
// m != nil，可以正常使用
```

### 删除与置零

```go
m := map[string]int{"a": 1, "b": 2}
delete(m, "a")  // 真正删除，释放内存
m["b"] = 0      // 值为 0，但 key 仍存在，不释放内存
```

**内存行为差异：**

| 操作         | len(m) | key 存在 | 内存释放 | tophash 标记       |
| :----------- | :----- | :------- | :------- | :----------------- |
| delete(m, k) | -1     | 否       | 是       | emptyOne/emptyRest |
| m[k] = 0     | 不变   | 是       | 否       | 不变               |

**delete 源码逻辑：**

```go
// 清理 key
if t.indirectkey() {
    *(*unsafe.Pointer)(k) = nil
} else if t.key.ptrdata != 0 {
    memclrHasPointers(k, t.key.size)
}
// 清理 value
if t.indirectelem() {
    *(*unsafe.Pointer)(e) = nil
} else if t.elem.ptrdata != 0 {
    memclrHasPointers(e, t.elem.size)
}
// 标记空位
b.tophash[i] = emptyOne
h.count--
```

**内存泄漏陷阱：**

```go
// 错误示例：不断置零而非删除
m := make(map[string]*Data)
for i := 0; i < 1000000; i++ {
    m[string(i)] = &Data{...}
}

// 错误：内存泄漏
for k := range m {
    m[k] = nil  // key 仍在，内存不释放
}

// 正确：真正删除
for k := range m {
    delete(m, k)  // key 删除，内存释放
}
```

**使用场景：**

- **使用 delete**：不再需要该 key，防止内存泄漏
- **使用置零**：key 需要保留，只是值需要清空

## 总结

| 特性       | 说明                          |
| :--------- | :---------------------------- |
| 底层结构   | hmap + bmap（桶）             |
| 桶容量     | 每桶 8 个键值对               |
| 解决冲突   | 链地址法（overflow 指针）     |
| 扩容方式   | 渐进式（每次操作迁移 1-2 桶） |
| 遍历顺序   | 随机（依赖实现不可靠）        |
| 并发安全   | 否（需要加锁）                |
| 查找复杂度 | 平均 O(1)，最差 O(n)          |

## 参考资料

- [map 的用法到 map 底层实现分析](https://blog.csdn.net/chenxun_2010/article/details/103768011)
- [Java map 和 golang map 的一些点](https://louyuting.blog.csdn.net/article/details/104238418)
- [深度解密 Go 语言之 map](https://www.cnblogs.com/qcrao-2018/archive/2019/05/22/10903807.html)
