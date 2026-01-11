---
date: "2026-01-11T18:54:20+08:00"
title: "Golang -- 单元测试"
tags: ["Golang", "Unit Test", "go test"]
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

单元测试（`Unit Testing`）是软件测试的基础层级，指针对程序中最小的可测试单元进行的独立验证，核心目标是确认单个单元的逻辑行为是否符合预期。

<!--more-->

特点:

1. 测试对象：最小可独立运行的代码单元（通常是函数 / 方法）。
2. 独立性：不依赖外部资源（如数据库、网络），通过模拟 / 桩函数隔离依赖，确保测试结果只由被测单元本身决定。
3. 自动化：可通过测试框架自动执行，无需手动干预。
4. 快速反馈：执行速度快，能在开发过程中即时发现代码缺陷。

Go 语言内置了 `go test` 这个一站式自动化测试工具，无需额外安装依赖，集成了 `单元测试`、`基准测试`、`示例测试`、`模糊 (Fuzz) 测试` 四大核心能力，还支持代码覆盖率分析、并行执行、精准筛选测试用例等高频功能。

本文对其中的单元测试进行说明介绍：

## 测试文件组织规则

1. 文件组织：测试文件与被测对象同包，或者独立存放在以 `_test` 结尾的包中。
2. 文件命名：测试文件必须以 `_test.go` 结尾。
3. 函数命名：测试函数必须以 `Test` 开头，且后续字母大写（如 TestAdd,）。
4. 函数签名：固定格式 `func TestXxx(*testing.T)`，参数 `*testing.T` 用于控制测试流程、报告错误。

## 基础示例

这里以求第 n 个斐波那契数为例，演示单元测试的基础用法：

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

## 子测试

测试时，一个单元（函数、方法）的测试用例（一组输入和预期输出就是一个用例）往往有多个，如果每一个都创建一个 `TestXxx` 函数，需要写大量重复的框架内容，也不利于管理，`testing.T` 中提供了 `Run` 方法来解决这一问题，使得我们可以将多个测试用例写在一个 `TestXxx` 中执行，并分别输出日志。

具体用法如下：

```golang
func TestFoo(t *testing.T) {
    // 这里可以写入公共的配置代码
    // <setup code>

    // 这里编写一个个测试用例
    t.Run("A=1", func(t *testing.T) { ... })
    t.Run("A=2", func(t *testing.T) { ... })
    t.Run("B=1", func(t *testing.T) { ... })

    // 如果需要回收清理资源，可以在这里进行
    // <tear-down code>
}
```

示例：
我们在其中故意添加了两个错误的用例，测试是否会将两个错误的用例都加入结果报告中：

```golang
func TestFibo(t *testing.T) {

	t.Run("1st fibo", func(t *testing.T) {
		n := 1
		want := 1

		got := Fibo(n)

		if got != want {
			t.Errorf("expected:%v, got:%v", want, got) // 测试失败输出错误提示
		}
	})

	t.Run("1st wrong", func(t *testing.T) {
		n := 1
		want := 0

		got := Fibo(n)

		if got != want {
			t.Errorf("expected:%v, got:%v", want, got) // 测试失败输出错误提示
		}
	})

	t.Run("3st fibo", func(t *testing.T) {
		n := 3
		want := 2

		got := Fibo(n)

		if got != want {
			t.Errorf("expected:%v, got:%v", want, got) // 测试失败输出错误提示
		}
	})

	t.Run("3st wrong", func(t *testing.T) {
		n := 3
		want := 3

		got := Fibo(n)

		if got != want {
			t.Errorf("expected:%v, got:%v", want, got) // 测试失败输出错误提示
		}
	})

}
```

运行结果如下：

```bash
go test
--- FAIL: TestFibo (0.00s)
    --- FAIL: TestFibo/1st_wrong (0.00s)
        fibo_unit_test.go:25: expected:0, got:1
    --- FAIL: TestFibo/3st_wrong (0.00s)
        fibo_unit_test.go:47: expected:3, got:2
FAIL
exit status 1
FAIL    fibo    0.001s
```

可以看到测试结果中既包含了出错的测试函数信息，也包含了所有异常用例信息。

## 基于表格的测试

可以看到即使使用了自测试，任然有大量重复代码，且用例分散任然不方便管理。上文中提到过每一个测试用例实际上是就是一个输入和输出的组合，我们完全可以使用表格的形式将它们组织到一起。

```golang
func TestFibo(t *testing.T) {
	tests := []struct {
		name string
		n    int
		want int
	}{
		{name: "1st fibo", n: 1, want: 1},
		{name: "1st wrong", n: 1, want: 0},
		{name: "3st fibo", n: 3, want: 2},
		{name: "3st wrong", n: 3, want: 5},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Fibo(tt.n)
			if got != tt.want {
				t.Errorf("Fibo() = %v, want %v", got, tt.want) // 测试失败输出错误提示
			}
		})
	}
}
```

这样组织测试用例，冗余代码减少了，用例更加清晰，添加修改用了也更加方便。

## 跳过测试

在某些时候我们可能希望跳过某些测试，`testing` 提供了 `T.Skip` 方法来达成这一目标。对于单元测试，我们通常会配合 `-short` 参数使用，用来跳过某些测试时间较长的用例。

```golang
func TestTimeConsuming(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping test in short mode.")
    }
    ...
}
```

其中 `testing.Short()` 会检测 `go test` 命令是否添加了 `-short` 参数，如果添加了则返回 `true`。

```bash
# 跳过耗时测试
go test -short
```

## TestMain

如果测试文件中有一些公共的前置操作或者后置操作需要执行，此时我们可以使用 `TestMain` 实现。  
如果测试文件包含函数 `func TestMain(m *testing.M)` 那么生成的测试会先调用 `TestMain(m)`，然后再通过 `m.Run` 运行具体测试。

> [!important] 需要注意的是：
>
> 1. 退出测试的时候应该使用 `m.Run` 的返回值作为参数调用 `os.Exit`
> 2. 使用 `TestMain` 时，`go test` 中的命令行参数不会再主动解析，如需使用，则需要手动执行解析。

```golang
func TestMain(m *testing.M) {
    flag.Parse() // 如果需要使用命令行参数 在这里执行 flag.Parse()
	fmt.Println("write setup code here...") // 测试之前的做一些设置
	// 如果 TestMain 使用了 flags，这里应该加上flag.Parse()
	retCode := m.Run()                         // 执行测试
	fmt.Println("write teardown code here...") // 测试之后做一些拆卸工作
	os.Exit(retCode)                           // 退出测试
}

```

## 并行测试

想要在单元测试过程中使用并行测试来提高测试性能，可以使用 `T.Parallel` 指明当前测试是否支持并行测试，如：

> [!important] 对于同一个 `*testing.T` 对象,`T.Parallel()` 不能执行多次。

```golang
func TestFibo(t *testing.T) {
	tests := []struct {
		name string
		n    int
		want int
	}{
		{name: "1st fibo", n: 1, want: 1},
		{name: "1st wrong", n: 1, want: 0},
		{name: "3st fibo", n: 3, want: 2},
		{name: "3st wrong", n: 3, want: 5},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
			got := Fibo(tt.n)
			if got != tt.want {
				t.Errorf("Fibo() = %v, want %v", got, tt.want) // 测试失败输出错误提示
			}
		})
	}
}
```

## 测试覆盖率

测试覆盖率是指代码被测试套件覆盖的百分比。通常我们使用的都是语句的覆盖率，也就是在测试中至少被运行一次的代码占总代码的比例。 
Go 提供内置功能来检查你的代码覆盖率，即使用 `go test -cover` 来查看测试覆盖率。

```bash
go test -cover
PASS
coverage: 100.0% of statements
ok      fibo    0.002s
```

Go 还提供了一个额外的 `-coverprofile` 参数，用来将覆盖率相关的记录信息输出到一个文件。

```bash
go test -cover -coverprofile=c.out
PASS
coverage: 100.0% of statements
ok      fibo    0.001s
```

我们可以使用 `go tool cover -html=c.out` 可视化的查看测试覆盖率报告文件。

## go test 参数

这一节补充一些常用的 `go test` 参数：

### -v

通过添加 `-v` 参数， `go test` 会打印更详细的测试报告，如：

```bash
=== RUN   TestFibo
=== RUN   TestFibo/1st_fibo
=== RUN   TestFibo/1st_wrong
    fibo_unit_test.go:30: Fibo() = 1, want 0
=== RUN   TestFibo/3st_fibo
=== RUN   TestFibo/3st_wrong
    fibo_unit_test.go:30: Fibo() = 2, want 5
--- FAIL: TestFibo (0.00s)
    --- PASS: TestFibo/1st_fibo (0.00s)
    --- FAIL: TestFibo/1st_wrong (0.00s)
    --- PASS: TestFibo/3st_fibo (0.00s)
    --- FAIL: TestFibo/3st_wrong (0.00s)
FAIL
exit status 1
FAIL    fibo    0.001s
```

### -run

通过 `-run` 参数我们可以灵活的指定需要执行的测试用例，一下是一些使用频率高的用法：

> 由于命令行中空格是用来分割元素的默认标识,所以对于子模块名包含空格的情况,会先将空格转换为 `_` 在进行匹配

1. 指定运行某一个测试函数：`-run=^测试函数名$` 如：`-run=^TestFibo$`
2. 指定前缀的测试函数：`-run=^函数前缀`，如：`-run=TestFibo` 执行前缀为 `TestFibo` 的测试函数
3. 运行目录下所有测试函数：`-run=.`
4. 运行子函数：通过 `/` 分割测试函数和自测试匹配模式，匹配模式同上。如：`-run=^TestFibo$/1st_wrong` 运行 `TestFibo` 中的前缀为 `1st_wrong` 的子测试

### -count

指定测试执行次数，主要作用是方便多次执行测试。

常通过添加命令行参数 `-count=1` 来避免 `go test` 使用缓存的测试结果。

`go test` 通过将需要执行的测试函数组织到自动生成的 `main.go` 文件中进行编译成可执行文件完成自动化测试的，在 `list package` 模式（未指明需要测试的包时） 如果发现生成的可执行文件没有变化且运行参数中不包含不可使用缓存的参数，`go test` 会优先使用缓存的执行结果。

## 参考资料

[Golnag 官方文档 -- testing 包](https://pkg.go.dev/testing)  
[Golang 中文学习文档 -- 测试](https://golang.halfiisland.com/essential/senior/120.test.html)  
[Go 单测从零到溜系列 0—单元测试基础](https://www.liwenzhou.com/posts/Go/unit-test-0/)
