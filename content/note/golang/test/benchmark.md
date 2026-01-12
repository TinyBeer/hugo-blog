---
date: "2026-01-12T09:55:57+08:00"
title: "Golang -- 基准测试"
tags: ["Golang", "Benchmark", "go test"]
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

基准测试（`Benchmark Testing`）是用于优化验证、瓶颈定位与选型决策。它是后端开发中评估代码、数据库、服务性能的核心手段。

<!--more-->

它通过标准化方法、工具和场景，对系统 / 组件的性能指标进行定量、可重复、可对比的测试，核心是建立性能基线，

`go test` 已经提供了基准测试能力，我们只需要按照一定规则编写测试文件即可。

基准测试的测试函数签名为 `func BenchmarkXxx(*testing.B)`，文件组织方式与之前介绍的单元测试相同，这里就不多赘述了。

## 简单示例

我们使用求第 n 个斐波那契数作为例子，这里提供了两种求解算法 递归 和 动态规划

```golang
package fibo

func Fibo(n int) int {
	if n <= 2 {
		return 1
	}
	return Fibo(n-1) + Fibo(n-2)
}

func Fibo2(n int) int {
	arr := make([]int, n)
	arr[0], arr[1] = 1, 1
	for i := 2; i < n; i++ {
		arr[i] = arr[i-2] + arr[i-1]
	}
	return arr[n-1]
}

```

测试代码：

测试代码使用了两种分格的代码 `b.N` 和 `b.Loop()`

```golang
func Benchmark_Fibo(b *testing.B) {
	for range b.N {
		Fibo(10)
	}
}

func Benchmark_Fibo2(b *testing.B) {
	for b.Loop() {
		Fibo2(10)
	}
}
```

执行测试：
`go test` 默认不会执行 基准测试函数 (Benchmark 为前缀的测试函数)，需要使用 `-bench` 参数开启基准测试，用法和 `-run` 参数一样，几个常用的模式：

1. 指定运行某一个测试函数：`-bench=^测试函数名$` 如：`-bench=^BenchFibo$`
2. 指定前缀的测试函数：`-bench=^函数前缀`，如：`-bench=BenchFibo` 执行前缀为 `BenchFibo` 的测试函数
3. 运行目录下所有测试函数：`-bench=.`

```bash
> go test -bench=.
goos: linux
goarch: amd64
pkg: fibo
cpu: Intel(R) Core(TM) i5-10400F CPU @ 2.90GHz
Benchmark_Fibo-12        7783959               155.5 ns/op
Benchmark_Fibo2-12      36437092                30.07 ns/op
PASS
ok      fibo    2.465s
```

测试报告中内容为每个测试函数执行次数，每次执行耗时。

## 并行测试

开启并行测试我们需要在测试代码中使用 `RunParallel`, 并配合 `go test` 的 `-cpu` 参数指明需要使用的 CPU 数量。

示例：

```golang
func Benchmark_Fibo_Parallel(b *testing.B) {
	b.RunParallel(func(p *testing.PB) {
		for p.Next() {
			Fibo(10)
		}
	})
}
```

执行：

`-bech` 参数说明测试 `Benchmark_Fibo_Parallel` 函数  
`-cpu` 参数指明使用 2,4,8 个 cpu 进行测试，测试报告中将不同 CPU 数量的执行结果通过后缀区分。

```bash
 > go test -bench=^Benchmark_Fibo_Parallel$ -cpu=2,4,8
goos: linux
goarch: amd64
pkg: fibo
cpu: Intel(R) Core(TM) i5-10400F CPU @ 2.90GHz
Benchmark_Fibo_Parallel-2       15328250                78.13 ns/op
Benchmark_Fibo_Parallel-4       26335464                38.52 ns/op
Benchmark_Fibo_Parallel-8       48672609                23.99 ns/op
PASS
ok      fibo    3.534s
```

## 子测试

使用 `b.Run` 开启自测试，用法和 单元测试 中一样，比较简单，这里就不进行说明了。

## b.N 和 b.Loop()

在 `b.Loop()` 之前，我们使用 `b.N` 风格的测试代码，他两比较直观的区别是使用 `b.Loop()` 仅记录循环体内的耗时，而 `b.N` 会将循环前的耗时统计入基准测试报告，为了确保测试准确，我们需要在循环前执行 `b.ResetTimer()` 来重置计时器。

官方更推荐使用 `b.Loop()` 风格的测试代码，因为它更加的健壮高效。

## go test 参数

这一节补充一些常用的 `go test` 参数：

### -benchmem

开启基准测试的内存分配统计

```bash
>go test -bench=. -benchmem
goos: linux
goarch: amd64
pkg: fibo
cpu: Intel(R) Core(TM) i5-10400F CPU @ 2.90GHz
Benchmark_Fibo-12                7781078               154.6 ns/op             0 B/op          0 allocs/op
Benchmark_Fibo2-12              39579192                28.81 ns/op           80 B/op          1 allocs/op
Benchmark_Fibo_Parallel-12      54490243                21.10 ns/op            0 B/op          0 allocs/op
PASS
ok      fibo    3.675s
```

新增的两列统计数据分别为内存分配量和内存分配次数

### -benchtime

通过 `-benchtime` 可以设置基准测试的最小时间单元，默认是 1s 。循环控制 `b.Loop()` 或 `b.N` 会持续循环到执行总时间超过最小时间单元后的退出。1s 时间对于某些耗时较长的测试是不够的，可能测试仅被执行几次就推出了。这是我们就需要用 `-benchtime` 参数了。

我们将测试内容改为求第 40 个斐波那契数：

```golang
func Benchmark_Fibo(b *testing.B) {
	for range b.N {
		Fibo(40)
	}
}
```

测试：

```bash
go test -bench=^Benchmark_Fibo$
goos: linux
goarch: amd64
pkg: fibo
cpu: Intel(R) Core(TM) i5-10400F CPU @ 2.90GHz
Benchmark_Fibo-12              4         290487535 ns/op
PASS
ok      fibo    2.318s
```

可以看到这是测试函数仅仅执行了 4 次，这样的样本数量显然是不够的，这时我们就可以通过 `-benchtime` 参数调整最小测试时间单元。

```bash
go test -bench=^Benchmark_Fibo$  -benchtime=20s
goos: linux
goarch: amd64
pkg: fibo
cpu: Intel(R) Core(TM) i5-10400F CPU @ 2.90GHz
Benchmark_Fibo-12             82         290069483 ns/op
PASS
ok      fibo    24.076s
```

### -count

设置基准测试执行多少轮，我们可以通过增加执行轮次提高测试精确度。

```bash
>go test -bench=. -count=2
goos: linux
goarch: amd64
pkg: fibo
cpu: Intel(R) Core(TM) i5-10400F CPU @ 2.90GHz
Benchmark_Fibo-12                7760100               153.9 ns/op
Benchmark_Fibo-12                7797336               153.7 ns/op
Benchmark_Fibo2-12              37978118                30.33 ns/op
Benchmark_Fibo2-12              42350168                28.70 ns/op
Benchmark_Fibo_Parallel-12      54041786                21.35 ns/op
Benchmark_Fibo_Parallel-12      51756321                21.39 ns/op
PASS
ok      fibo    7.384s
```

<!-- Todo golang.org/x/perf/benchstat -->

## 参考资料

[Golnag 官方文档 -- testing 包](https://pkg.go.dev/testing)  
[Golang 中文学习文档 -- 测试](https://golang.halfiisland.com/essential/senior/120.test.html)  
[Go 单测从零到溜系列 0—单元测试基础](https://www.liwenzhou.com/posts/Go/unit-test-0/)
