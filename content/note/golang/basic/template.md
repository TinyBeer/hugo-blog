---
date: "2025-12-29T22:32:37+08:00"
title: "Golang -- Template 模板渲染"
tags: ["Golang", "Template"]
categories: "笔记"
# description: "Desc Text."
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

`template` 实现了一种基于数据驱动的文本生成模板。`Golang` 提供了两个包 `text/template` 和 `html/template`。

<!--more-->

我们通过传入数据（通常是结构体或者 Map）驱动模板，模板通过注释指明需要使用的数据元素，从而渲染出想要的结果。

`text/template` 假设作者（数据传入方）是可以信任的，从而允许更自由的渲染。而 `text/template` 出于安全考虑，会对输出进行转义，从而避免一些注入攻击。
模板可以分成两部分，一部分是由 "{{" 和 "}}" 包裹的 `Action`（动态文本），另一部分为静态文本，会原封不动地被渲染。

另外，`template` 包是支持并行渲染的。

下面是一个简单的例子：

```golnag
package main

import (
	"html/template"
	"os"
)

type Inventory struct {
	Material string
	Count    uint
}

func main() {
	sweaters := Inventory{"wool", 17}
	tmpl, err := template.New("test").Parse("{{.Count}} items are made of {{.Material}}")
	if err != nil {
		panic(err)
	}
	err = tmpl.Execute(os.Stdout, sweaters)
	if err != nil {
		panic(err)
	}
}
```

执行结果如下：

```bash
go run .
17 items are made of wool%
```

## 文本和空格

默认情况下，所有 `Action` 以外的文本和空格会被原封不动地拷贝。当然我门可以通过 `-` 和空格指定移除分隔符一侧的空格，`{{-` 移除左侧空格，`-}}` 移除右侧空格，如：

> 注意：
> 分隔符和 `-` 之间不能有空格，`-` 和 `Action` 中的内容需要有空格分割。  
> 反例： `{{-3}}` 会被渲染为 `-3`，而不是移除左侧空格后的 `3`。

```tpl
"{{23 -}} < {{- 45}}"
```

渲染结果如下：

```bash
"23<45"
```

## Action 动作

### 注释动作

```tpl
{{/* 注释内容 */}}
{{- /* 注释内容 */ -}}
```

1. 基础注释：
   语法：`/*` 和 `*/` 是注释界定符

   ```tpl
   {{/* 注释内容 */}}
   ```

   特点：注释前后的空白字符（空格、制表符等）会保留，仅注释内容本身被丢弃

2. 去除空格的注释
   语法：在基础注释的 `{{` 后加 `-`、`}}` 前加 `-`。

   ```tpl
   {{- /* 注释内容 */ -}}
   ```

   核心作用：自动修剪注释前后的空白字符（包括空格、换行符、制表符等），避免模板渲染后出现多余的空白行或空格，让输出更整洁

3. 补充说明：
   注释支持换行（可跨多行编写），但不支持嵌套

### 管道输出动作

```tpl
{{pipeline}}
{{.Age}}
{{.Name | upper}}
{{len(.Name)}}
```

这是 Go 模板最基础、最常用的动作，作用是将 `管道（pipeline）` 的取值以默认文本格式输出到最终结果中：

1. `管道（pipeline）`的含义：
   可以是模板的上下文变量（最常用的是 `.`，代表当前作用域的根数据；也可以是数据的字段，如 `.Name`、`.Age`，对应 Go 结构体的字段或 map 的 key）、函数调用（如 len(.List)）、多个操作的链式组合（如 `.Name | upper`，将名称转为大写后输出）
2. 输出格式：
   默认使用 fmt.Print 函数的格式化规则（即 Go 语言的默认文本输出格式），无需手动指定格式，直接渲染变量或计算结果

### 条件判断动作

```tpl
{{if pipeline}} T1 {{end}}
{{if pipeline}} T1 {{else}} T0 {{end}}
{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}
```

类似 `Go` 语言的 `if-else` 逻辑，根据`管道（pipeline）`的值是否为 `零值`，决定执行哪一段模板内容，且不会修改上下文变量。`Go` 模板定义的 `零值` 包括：`false`（布尔值）、0（整数）、`nil` 指针 / 接口、长度为 `0` 的 数组 / 切片 / map / 字符串`。

1. 单分支判断：  
   `{{if pipeline}} T1 {{end}}`  
   逻辑：如果 `pipeline` 的值非零值，执行 `T1` 段模板（渲染 `T1` 内容）；如果 `pipeline` 的值为零值，不输出任何内容
2. 双分支判断：  
   `{{if pipeline}} T1 {{else}} T0 {{end}}`  
   逻辑：如果 `pipeline` 的值非零值，执行 `T1`；如果为零值，执行 `T0`（二选一执行）
3. 多分支判断：  
   `{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}`  
   逻辑：简化 `if-else` 嵌套的写法，等价于  
   `{{if pipeline}} T1 {{else}}{{if pipeline}} T0 {{end}}{{end}}`  
   特点：支持连续写多个 `else if`，按顺序判断，满足第一个非空条件后执行对应模板段，后续条件不再判断

### 循环遍历动作

```tpl
{{range pipeline}} T1 {{end}}
{{range pipeline}} T1 {{else}} T0 {{end}}

{{break}}
{{continue}}
```

类似 `Go` 语言的 `for range` 循环，用于遍历数组、切片、map、整数、通道（channel）、`iter.Seq/iter.Seq2`（Go 1.23+ 迭代器）等集合类型数据，核心特性是遍历过程中会修改上下文变量。

1. 基础循环：  
   `{{range pipeline}} T1 {{end}}`  
   前置要求：`pipeline` 必须是上述支持遍历的类型  
   逻辑：如果 `pipeline` 的长度为 0（空集合 / 无元素），不输出任何内容；否则，依次将 `.` 设为集合中的每个元素，对每个元素执行一次 `T1` 模板段
   特殊说明：遍历 map 时，如果 key 是 `Go` 基础类型（int、string 等）且有默认排序规则，会按 `key` 升序遍历；否则按随机顺序遍历
2. 带空分支的循环：  
   `{{range pipeline}} T1 {{else}} T0 {{end}}`  
   逻辑：如果 `pipeline` 长度为 0（空集合），不修改 `.`，执行 `T0` 模板段；否则，按基础循环逻辑执行 `T1`
   循环控制：  
   `{{break}} 和 {{continue}}`  
   `{{break}}`：立即停止当前迭代，且跳出最内层的 `{{range}}` 循环
   `{{continue}}`：进入最内层 `{{range}}` 循环的下一个元素的迭代
   注意：仅能在 `{{range}}` 循环内部使用，不能用于其他场景

### 模板复用动作

```tpl
{{template "name"}}
{{template "name" pipeline}}
{{block "name" pipeline}} T1 {{end}}
```

`Go` 模板支持 `模板复用`，将常用的模板片段定义为独立模板，然后在主模板中引用，减少重复代码，提升可维护性。

1. 基础模板引用：  
	`{{template "name"}}`  
	逻辑：执行名为 `name` 的模板 ( `{{define "name"}}` 定义的模板) ，且执行时传入的数据为 `nil`  
	适用场景：无依赖外部数据的公共片段  
2. 带数据的模板引用：  
	`{{template "name" pipeline}}`
	逻辑：执行名为 `name` 的模板，且将 `pipeline` 的取值作为该模板的上下文数据  
3. 块模板：  
   `{{block "name" pipeline}} T1 {{end}}`  
	本质：语法糖，等价于 `先定义模板，再立即引用该模板`，分为两步：  
	1. 隐式定义模板：`{{define "name"}} T1 {{end}}`  
	2. 立即执行模板：`{{template "name" pipeline}}`  
	核心用途：定义 `根模板`（如 HTML 基础骨架），后续可通过重新定义 `name` 模板，实现对根模板的自定义扩展`（类似「模板继承」的效果）`  
	适用场景：统一页面布局，仅替换局部内容`（如所有页面共用头部和尾部，仅正文部分自定义）`

### 上下文切换动作
```tpl
{{with pipeline}} T1 {{end}}
{{with pipeline}} T1 {{else}} T0 {{end}}
{{with pipeline}} T1 {{else with pipeline}} T0 {{end}}
```
`{{with}}` 动作的核心作用是临时切换模板的上下文根数据，简化对深层数据的访问，且支持分支判断（基于数据是否为零值）。

1. 基础上下文切换：  
   `{{with pipeline}} T1 {{end}}`  
	逻辑：如果 `pipeline` 的值为零值，不输出任何内容；如果 pipeline 的值非零值，将上下文根数据临时设为 `pipeline` 的值，在 `T1` 模板段内，所有 `.XXX` 均基于该临时数据访问，退出 `{{end}}` 后，恢复为原来的上下文数据
2. 带空分支的上下文切换：  
   `{{with pipeline}} T1 {{else}} T0 {{end}}`  
	逻辑：如果 `pipeline` 的值非零值，切换上下文并执行 `T1`；如果 `pipeline` 的值为空，不修改上下文，执行 `T0`
3. 多分支上下文切换：  
   `{{with pipeline}} T1 {{else with pipeline}} T0 {{end}}`  
	逻辑：简化 `{{with}}` 的嵌套写法，等价于 `{{with pipeline}} T1 {{else}}{{with pipeline}} T0 {{end}}{{end}}`  
	特点：按顺序判断多个 `pipeline`，第一个非零的 `pipeline` 会切换上下文并执行对应模板段，后续分支不再判断

### Arguments 变量

## 参考资料

[text/template](https://pkg.go.dev/text/template)  
[html/template](https://pkg.go.dev/html/template)
