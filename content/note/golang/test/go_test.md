---
date: "2026-01-11T15:44:52+08:00"
title: "Golang -- 测试简介"
tags: ["Golang", "go test"]
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

`go test` 是 `Go` 语言内置的一站式自动化测试工具，无需额外安装依赖，集成了 `单元测试`、`基准测试`、`示例测试`、`模糊 (Fuzz) 测试` 四大核心能力，还支持代码覆盖率分析、并行执行、精准筛选测试用例等高频功能。

<!--more-->

## 测试文件组织

使用 `go test` 进行自动化测试，需要按照一定的规则编排测试文件：

1. 测试文件  
   测试文件需是以 `_test.go` 为后缀的 go 文件，满足 `*_test.go` 匹配规则。
2. 测试包
   测试文件可以同被测试内容在同一包内，也可以使用独立包管理。  
   测试文件和被测试对象在同一包中，可以测试未导出内容。  
   而使用独立包管理测试文件，则需要使用 `_test` 作为包名后缀，测试时需要将被测试包导入（import），且不能访问为导出的内容。

## 测试函数

测试函数需要放在测试文件（`*_test.go`）内统一管理，类型及格式规则说明如下：

> 其中 `Xxx` 表示以非小写字母开头的内容

| 测试类型     | 函数签名                       | 作用                           |
| :----------- | :----------------------------- | :----------------------------- |
| 单元测试函数 | func TestXxx(\*testing.T)      | 确保代码的逻辑正确性           |
| 基准测试函数 | func BenchmarkXxx(\*testing.B) | 测试代码性能                   |
| 实例测试函数 | func ExampleXxx()              | 提供代码示例、确保示例代码正确 |
| 模糊测试函数 | func FuzzXxx(\*testing.F)      | 探测代码在异常输入中的 bug     |

## 单元测试函数

下面使用求第 n 个斐波那契数作为示例，演示单元测试的使用方法：

```golang
// fibo.go
package fibo

func Fibo(n int) int {
	if n <= 2 {
		return 1
	}
	return Fibo(n-1) + Fibo(n-2)
}
```

测试文件：

```golang
// fibo_test.go
package fibo

import "testing"

func TestFibo1(t *testing.T) {
	n := 1
	want := 1

	got := Fibo(n)

	if got != want {
		t.Errorf("expected:%v, got:%v", want, got) // 测试失败输出错误提示
	}
}

func TestFibo3(t *testing.T) {
	n := 3
	want := 2

	got := Fibo(n)

	if got != want {
		t.Errorf("expected:%v, got:%v", want, got) // 测试失败输出错误提示
	}
}
```

在当前目录执行 `go test` 可以看到类似如下结果:

```bash
go test
PASS
ok      fibo    0.001s
```

## 基准测试

我们需要为 `go test` 命令添加 `-bench` 参数开启基准测试，参数一般使用以下几种设置方式：

1. 执行单个基准测试
   `-bench=^测试函数名$`，如： `-bench=^BenchmarkFibo$`
2. 执行匹配前缀的所有基准测试
   `-bench=测试函数前缀`，如：`-bench=BenchmarkFibo`
3. 执行当前路径下所有基准测试
   `-bench=.`

测试文件：

```golang
package fibo

import "testing"

func BenchmarkFibo(b *testing.B) {
	for b.Loop() {
		Fibo(9)
	}
}
```

执行测试：

```bash
go test  -bench=^BenchmarkFibo$
goos: linux
goarch: amd64
pkg: fibo
cpu: 12th Gen Intel(R) Core(TM) i5-12400
BenchmarkFibo-12        14825277                81.01 ns/op
PASS
ok      fibo    1.203s
```

## 示例测试

示例测试要求编写测试时将需要验证的内容通过 `fmt.Println` 打印，然后通过在函数末尾添加注释，指明预期结果。目前支持两种预期结果：

1. `// Output:` 有序匹配
   预期结果按顺序通过注释排列在 `// Output:` 后

   ```golang
   func ExampleHello() {
   	fmt.Println("hello")
   	// Output: hello
   }

   func ExampleSalutations() {
   	fmt.Println("hello, and")
   	fmt.Println("goodbye")
   	// Output:
   	// hello, and
   	// goodbye
   }
   ```

2. `// Unordered output:` 无序匹配
   允许预期结果是无需输出的，预期结果同样以注释方式排列在后方

   ```golang
   func ExamplePerm() {
   	for _, value := range Perm(5) {
   		fmt.Println(value)
   	}
   	// Unordered output: 4
   	// 2
   	// 1
   	// 3
   	// 0
   }
   ```

测试文件：

```golang
package fibo

import "fmt"

func ExampleFibo() {
	num := Fibo(9)
	fmt.Println(num)
	// Output:
	// 34

}

```

执行 `go test` 可以看到类似如下结果:

```bash
go test
PASS
ok      fibo    0.001s
```

## 模糊测试

`Fuzz 测试（模糊测试）`，是 `Go 1.18` 版本正式内置到标准库 `testing` 包中的自动化随机测试技术，无需任何第三方依赖，和单元测试 / 基准测试一样，通过 `go test` 命令直接运行。  
测试的核心逻辑：自动生成海量「随机、畸形、边界值、非常规组合」的输入数据，持续传入被测函数，主动触发程序的 `panic`、数组越界、空指针、数值溢出、逻辑异常 等隐藏问题；  
一旦发现程序崩溃 / 异常，会自动保存「触发崩溃的输入数据」，方便开发者复现和修复 `BUG`。

这里给出一个简单的示例：

```golang
package fibo

import (
	"bytes"
	"encoding/hex"
	"testing"
)

func FuzzHex(f *testing.F) {
	for _, seed := range [][]byte{{}, {0}, {9}, {0xa}, {0xf}, {1, 2, 3, 4}} {
		f.Add(seed)
	}
	f.Fuzz(func(t *testing.T, in []byte) {
		enc := hex.EncodeToString(in)
		out, err := hex.DecodeString(enc)
		if err != nil {
			t.Fatalf("%v: decode: %v", in, err)
		}
		if !bytes.Equal(in, out) {
			t.Fatalf("%v: not equal after round trip: %v", in, out)
		}
	})
}

```

执行模糊测试需要使用 `-fuzz` 参数，参数格式可以类比基准测试：

1. 执行单个模糊测试
   `-fuzz=^测试函数名$`，如： `-fuzz=^FuzzHex$`
2. 执行匹配前缀的所有模糊测试
   `-fuzz=测试函数前缀`，如：`-fuzz=Fuzz`
3. 执行当前路径下所有模糊测试
   `-fuzz=.`

> [!importance] 模糊测试默认是一直运行的，通过 `Ctrl+C` 停止，如需控制执行时间，可以通过 `-fuzztime` 设置运行时长。

```bash
go test -fuzz=^FuzzHex
fuzz: elapsed: 0s, gathering baseline coverage: 0/6 completed
fuzz: elapsed: 0s, gathering baseline coverage: 6/6 completed, now fuzzing with 12 workers
fuzz: elapsed: 3s, execs: 1350167 (449957/sec), new interesting: 6 (total: 12)
fuzz: elapsed: 6s, execs: 2704690 (451481/sec), new interesting: 6 (total: 12)
fuzz: elapsed: 9s, execs: 4080644 (458663/sec), new interesting: 6 (total: 12)
^Cfuzz: elapsed: 11s, execs: 4851936 (446388/sec), new interesting: 6 (total: 12)
PASS
ok      fibo    10.731s
```

## 参考资料

[Golnag 官方文档 -- testing 包](https://pkg.go.dev/testing)  
[Golang 中文学习文档 -- 测试](https://golang.halfiisland.com/essential/senior/120.test.html)  
[Go 单测从零到溜系列 0—单元测试基础](https://www.liwenzhou.com/posts/Go/unit-test-0/)
