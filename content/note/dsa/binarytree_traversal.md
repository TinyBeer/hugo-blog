---
date: "2021-04-30T00:00:00+08:00"
title: "二叉树的遍历总结"
tags: ['BinaryTree', 'DSA']
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

# 二叉树的遍历

二叉树是编程中最常见的数据结构之一，由于其结构清晰、访问效率高等特点被广泛应用。本文将从时间复杂度和空间复杂度两个层面，对二叉树的遍历进行总结。主要包括：

- 利用系统栈遍历
- 利用用户栈遍历
- 二叉线索树遍历
- Morris 算法遍历
- 层序遍历（BFS）

> **约定**：下文空间复杂度中的 h 表示树的高度。平衡树 h = O(log n)，退化为链表时 h = O(n)。

## 准备工作

### 结点结构

```go
// the node of a binary tree
type TreeNode struct {
    Val int
    LChild, RChild *TreeNode
}
```

### 测试用树

```go
// GetTree 返回如下二叉树
//        1
//       / \
//      2   3
//     / \ / \
//    4  5 6  7
func GetTree() *TreeNode {
    return &TreeNode{
        Val: 1,
        LChild: &TreeNode{
            Val: 2,
            LChild: &TreeNode{
                Val: 4,
            },
            RChild: &TreeNode{
                Val: 5,
            },
        },
        RChild: &TreeNode{
            Val: 3,
            LChild: &TreeNode{
                Val: 6,
            },
            RChild: &TreeNode{
                Val: 7,
            },
        },
    }
}
```

## 利用系统栈遍历

使用系统栈进行遍历最为简单，此处提供了前序中序后序遍历代码：

### 前中后序遍历代码

```go
const (
    PRE = iota // 先序（前序）
    IN         // 中序
    POST       // 后序
)

// 使用系统栈进行遍历
func TraversalWithSystemStack(root *TreeNode, order int) {
    if root == nil {
        return
    }

    if order == PRE { // 先序
        fmt.Print(root.Val, "->")
    }
    TraversalWithSystemStack(root.LChild, order)
    if order == IN { // 中序
        fmt.Print(root.Val, "->")
    }

    TraversalWithSystemStack(root.RChild, order)
    if order == POST { // 后序
        fmt.Print(root.Val, "->")
    }
}
```

### 算法复杂度分析

时间复杂度 O(n)，空间复杂度 O(h)

## 利用用户栈遍历

利用用户栈进行遍历，可以避免系统栈中程序上下文信息开销，对内存占用有一定的优化作用。

### 先序遍历代码

```go
func PreOrderWithUserStack(root *TreeNode) {
    if root == nil {
        return
    }
    stack := []*TreeNode{root}
    for len(stack) > 0 {
        // 弹栈
        n := len(stack) - 1
        node := stack[n]
        stack = stack[:n]
        fmt.Print(node.Val, "->")
        // 注意压栈顺序：先右后左
        if node.RChild != nil {
            stack = append(stack, node.RChild)
        }
        if node.LChild != nil {
            stack = append(stack, node.LChild)
        }
    }
}
```

### 中序遍历代码

```go
func InOrderWithUserStack(root *TreeNode) {
    stack := []*TreeNode{}
    cur := root
    for cur != nil || len(stack) > 0 {
        // 一路向左，全部压栈
        for cur != nil {
            stack = append(stack, cur)
            cur = cur.LChild
        }
        // 弹栈，访问，转向右子树
        n := len(stack) - 1
        node := stack[n]
        stack = stack[:n]
        fmt.Print(node.Val, "->")
        cur = node.RChild
    }
}
```

### 后序遍历代码

```go
func PostOrderWithUserStack(root *TreeNode) {
    if root == nil {
        return
    }
    stack := []*TreeNode{root}
    var result []int
    for len(stack) > 0 {
        n := len(stack) - 1
        node := stack[n]
        stack = stack[:n]
        result = append(result, node.Val)
        // 后序：先压左再压右，弹出时反过来
        if node.LChild != nil {
            stack = append(stack, node.LChild)
        }
        if node.RChild != nil {
            stack = append(stack, node.RChild)
        }
    }
    // 反转输出
    for i := len(result) - 1; i >= 0; i-- {
        fmt.Print(result[i], "->")
    }
}
```

### 算法复杂度分析

时间复杂度 O(n)，空间复杂度 O(h)

## 二叉线索树遍历

下图为二叉搜索树的结点结构。

```
左子树或前驱结点  左标志位  值  右标志位  右子树或后继结点
    LChild        LTag    Val   RTag       RChild
```

当 LTag == 0 时，LChild 指向左子树；当 LTag == 1 时，LChild 指向中序前驱。RTag 同理。

### 线索化代码

```go
type ThreadNode struct {
    Val                  int
    LChild, RChild       *ThreadNode
    LTag, RTag           int // 0: child, 1: thread
}

var pre *ThreadNode // 中序遍历的前一个节点

// 中序线索化
func InThread(node *ThreadNode) {
    if node == nil {
        return
    }
    InThread(node.LChild)
    if node.LChild == nil {
        node.LTag = 1
        node.LChild = pre
    }
    if pre != nil && pre.RChild == nil {
        pre.RTag = 1
        pre.RChild = node
    }
    pre = node
    InThread(node.RChild)
}

// 中序遍历线索树（无需栈）
func InOrderThread(root *ThreadNode) {
    node := root
    for node != nil {
        // 找到最左节点
        for node.LTag == 0 {
            node = node.LChild
        }
        fmt.Print(node.Val, "->")
        // 沿后继遍历
        for node.RTag == 1 && node.RChild != nil {
            node = node.RChild
            fmt.Print(node.Val, "->")
        }
        node = node.RChild
    }
}
```

### 算法复杂度

时间复杂度 O(n)，空间复杂度 O(1)

## Morris 算法遍历

使用 Morris 算法进行遍历可以在保持遍历时间复杂度为 O(n)的情况下，将空间复杂度优化到 O(1)，实现树的前中后序遍历。

核心思想：利用叶子节点的空指针建立临时线索，遍历完成后恢复树的原始结构。

### 先序遍历代码

```go
func PreOrderMorris(root *TreeNode) {
    cur := root
    for cur != nil {
        mostright := cur.LChild
        if mostright != nil {
            for mostright.RChild != nil && mostright.RChild != cur {
                mostright = mostright.RChild
            }
            if mostright.RChild == nil {
                // 第一次访问：建立线索，输出当前节点
                mostright.RChild = cur
                fmt.Print(cur.Val, "->")
                cur = cur.LChild
                continue
            } else {
                // 第二次访问：恢复树结构
                mostright.RChild = nil
            }
        } else {
            // 无左子树：直接输出
            fmt.Print(cur.Val, "->")
        }
        cur = cur.RChild
    }
}
```

### 中序遍历代码

```go
func InOrderMorris(root *TreeNode) {
    cur := root
    for cur != nil {
        mostright := cur.LChild
        if mostright != nil {
            for mostright.RChild != nil && mostright.RChild != cur {
                mostright = mostright.RChild
            }
            if mostright.RChild == nil {
                // 第一次访问：建立线索
                mostright.RChild = cur
                cur = cur.LChild
                continue
            } else {
                // 第二次访问：恢复结构，输出当前节点
                mostright.RChild = nil
            }
        }
        // 无左子树，或从左子树返回：输出当前节点
        fmt.Print(cur.Val, "->")
        cur = cur.RChild
    }
}
```

### 后序遍历代码

```go
func PostOrderMorris(root *TreeNode) {
    if root == nil {
        return
    }
    dump := &TreeNode{Val: 0, LChild: root}
    cur := dump
    for cur != nil {
        mostright := cur.LChild
        if mostright != nil {
            for mostright.RChild != nil && mostright.RChild != cur {
                mostright = mostright.RChild
            }
            if mostright.RChild == nil {
                mostright.RChild = cur
                cur = cur.LChild
                continue
            } else {
                mostright.RChild = nil
                // 第二次访问：逆序输出左子树最右路径
                printReverse(cur.LChild)
            }
        }
        cur = cur.RChild
    }
}

// 逆序输出链表（用于后序遍历）
func printReverse(node *TreeNode) {
    tail := reverseList(node)
    cur := tail
    for cur != nil {
        fmt.Print(cur.Val, "->")
        cur = cur.LChild
    }
    reverseList(tail)
}

// 反转链表
func reverseList(node *TreeNode) *TreeNode {
    var prev *TreeNode
    cur := node
    for cur != nil {
        next := cur.RChild
        cur.RChild = prev
        prev = cur
        cur = next
    }
    return prev
}
```

### 算法复杂度分析

时间复杂度 O(n)，空间复杂度 O(1)

## 层序遍历（BFS）

层序遍历使用队列，按层从左到右访问每个节点。

### 基本层序遍历

```go
func LevelOrder(root *TreeNode) {
    if root == nil {
        return
    }
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        fmt.Print(node.Val, "->")
        if node.LChild != nil {
            queue = append(queue, node.LChild)
        }
        if node.RChild != nil {
            queue = append(queue, node.RChild)
        }
    }
}
```

### 按层分组输出

```go
func LevelOrderGrouped(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var result [][]int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, 0, size)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.LChild != nil {
                queue = append(queue, node.LChild)
            }
            if node.RChild != nil {
                queue = append(queue, node.RChild)
            }
        }
        result = append(result, level)
    }
    return result
}
```

### 算法复杂度分析

时间复杂度 O(n)，空间复杂度 O(n)（最坏情况最后一层有 n/2 个节点）

## 遍历的应用场景

| 遍历方式 | 典型应用 |
| :--- | :--- |
| 前序 | 复制/序列化二叉树、获取前缀表达式 |
| 中序 | BST 的有序输出、验证 BST 合法性 |
| 后序 | 计算表达式树、删除/释放二叉树、获取后缀表达式 |
| 层序 | 按层处理、求树的宽度、最短路径（无权图） |
| Morris | 内存受限场景下的 O(1) 空间遍历 |

## 算法比较

| | 时间复杂度 | 空间复杂度 | 特点 |
| :--- | :--- | :--- | :--- |
| 系统栈 | O(n) | O(h) | 最简单，递归实现 |
| 用户栈 | O(n) | O(h) | 避免系统栈开销，可控 |
| 线索树 | O(n) | O(1) | 预处理后遍历无需栈 |
| Morris | O(n) | O(1) | 无需预处理，临时修改树结构 |
| 层序 | O(n) | O(n) | 按层访问，使用队列 |

## 测试

使用 go test 测试结果

```
goos: windows
goarch: amd64
pkg: Algorithm/TraversalOfTree
cpu: AMD Ryzen 7 4800U with Radeon Graphics
BenchmarkTraversalWithSystemStack  1000000  1021 ns/op  0 B/op  0 allocs/op
BenchmarkPreOrderWithUserStack     1000000  1164 ns/op  0 B/op  0 allocs/op
BenchmarkPreOrderMorris            1494206   815.2 ns/op  0 B/op  0 allocs/op
PASS
ok  Algorithm/TraversalOfTree  4.203s
```
