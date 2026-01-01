---
date: "2025-12-29T22:32:37+08:00"
title: "Golang -- Template 模板渲染"
tags: ["Golang", "Template"]
categories: "笔记"
# description: "Desc Text."
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

## Arguments 参数

参数是一个简单的值，它可以是以下任意一个：

1. Go 语言中的常量，包括布尔值、字符串、字符、整形、浮点型、虚数、复数。
   值得注意的是，该值是否益处同 Go 语言中的常量一样取决于主机的 int 是 32 位还是 64 位。
2. 关键字 `nil`
3. 字符 `.`
   代表 `dot` 的值
4. 变量名
   以 `$`开始，后可接字母数字组成的字符串。单独的 `$` 也是合法的。
5. 结构体的字段名
   以 `.` 开头，被去值的数据需是一个结构体。支持链式取值，也可以从一个变量中取值，如：
   ```tpl
   .Field
   .Field1.Field2
   $x.Field1.Field2
   ```
   注意：结构体字段名需要为大写字母开头，才可以进行取值。
6. `Map` 的 `Key`
   以 `.` 开头，被去值的数据需是一个 `Map`。同样持链式取值，也可以从一个变量中取值，如：
   ```tpl
   .Key
   .Field1.Key1.Field2.Key2
   $x.key1.key2
   ```
   区别于结构体字段名，`Map` 不需要使用大写字母开头。
7. 方法名
   以 `.` 开头，方法可以为空。当需要调用该方法时（如：`dot.Method()`），需要保证方法的返回值只有一下两种情况：

   1. 一个返回值
   2. 两个返回值，第二个值为 `error`。当第二个返回值非零值时，渲染过程会终止，`error`会被返回给渲染的执行者。

   方法的使用也同样灵活，一下形式都是可以的：

   ```tpl
   .Method
   .Field1.Key1.Method1.Field2.Key2.Method2
   $x.Method1.Field
   ```

8. 函数名 `func`  
   返回值要求同方法一样，详情[参考](#函数)后文。

9. 由括号包裹的以上任意一个实体

   括号的作用是进行分组。它的结果同样可以进行结构体字段取值、`Map` 键值映射。

   ```tpl
   print (.F1 arg1) (.F2 arg2) (.StructValuedMethod "arg").Field
   ```

参数可以是为任何类型；如果它们是指针，则实现会在需要时自动指向基类型。如果是函数，例如结构的函数值字段，则该函数不会自动调用，但可以用作 `If` 动作等的真值。要调用它，请使用下面定义的 `call` 函数。

## 管道

管道可能是一系列“命令”的链式序列。命令是一个简单的值（参数）或函数或方法调用，可能有多个参数。  
通过用管道字符 `|` 分隔命令序列，可以将管道“链接”起来。这里可以类比终端中的管道运算符 `|`，作用是将上一个命令的结果作为下一个命令的最后一个参数使用，从而实现链式命令执行。  
管道的结果也同方法和函数的要一样。

## 变量

1. 变量的初始化

   ```tpl
   $variable := pipeline
   ```

2. 变量的赋值

   ```tpl
   $variable = pipeline
   ```

3. 在循环遍历(`range`)动作中使用变量

   ```tpl
   range $index, $element := pipeline
   ```

   这里变量的赋值可以类比 `Go语言` 中 `range` 语法。当 `pipeline` 为数组或者切片时候，`$index`, `$element` 为索引和元素。当 `pipeline` 为 `Map` 时，`$index`, `$element` 为键和元素。

变量的作用域延伸到声明它的控制结构的 `end` 操作（`if`、`with` 或 `range`），或者如果没有这样的控制结构，则延伸到模板的末尾。模板调用不会从其调用点继承变量。

## 函数

函数存在于两个函数表中，`模板函数表` 和 `全局函数表`
在执行过程时，首先在 `模板函数表` 寻找，然后在 `全局函数表` 中。  
默认情况下，模板中没有定义函数，但可以使用 `Funcs` 方法添加它们。

### 全局函数

以下是预定义的全局函数：`

```plaintext`
and
返回第一个零值参数或者最后一个参数。

    Returns the boolean AND of its arguments by returning the
    first empty argument or the last argument. That is,
    "and x y" behaves as "if x then y else x."
    Evaluation proceeds through the arguments left to right
    and returns when the result is determined.

call
执行函数，第一个参数为要执行的函数，后续参数将作为函数的入参。

    Returns the result of calling the first argument, which
    must be a function, with the remaining arguments as parameters.
    Thus "call .X.Y 1 2" is, in Go notation, dot.X.Y(1, 2) where
    Y is a func-valued field, map entry, or the like.
    The first argument must be the result of an evaluation
    that yields a value of function type (as distinct from
    a predefined function such as print). The function must
    return either one or two result values, the second of which
    is of type error. If the arguments don't match the function
    or the returned error value is non-nil, execution stops.

html
对入参进行 HTML 转义。

    Returns the escaped HTML equivalent of the textual
    representation of its arguments. This function is unavailable
    in html/template, with a few exceptions.

index
索引 数组、切片或者 map。

    Returns the result of indexing its first argument by the
    following arguments. Thus "index x 1 2 3" is, in Go syntax,
    x[1][2][3]. Each indexed item must be a map, slice, or array.

slice
切片操作，对数组或者切片取切片。

    slice returns the result of slicing its first argument by the
    remaining arguments. Thus "slice x 1 2" is, in Go syntax, x[1:2],
    while "slice x" is x[:], "slice x 1" is x[1:], and "slice x 1 2 3"
    is x[1:2:3]. The first argument must be a string, slice, or array.

js
转义 Js 代码。

    Returns the escaped JavaScript equivalent of the textual
    representation of its arguments.

len
取长度。

    Returns the integer length of its argument.

not
非。
Returns the boolean negation of its single argument.

or
或。

    Returns the boolean OR of its arguments by returning the
    first non-empty argument or the last argument, that is,
    "or x y" behaves as "if x then x else y".
    Evaluation proceeds through the arguments left to right
    and returns when the result is determined.

print
同 fmt.Sprint

    An alias for fmt.Sprint

printf
同 fmt.Sprintf

    An alias for fmt.Sprintf

println
同 fmt.Sprintln

    An alias for fmt.Sprintln

urlquery
URL 转义。

    Returns the escaped value of the textual representation of
    its arguments in a form suitable for embedding in a URL query.
    This function is unavailable in html/template, with a few
    exceptions.

````

> 布尔函数将任何零值视为 false，将非零值视为 true。

比较函数：
比较函数适用于所有 `Go语言` 中的可比较类型，对于整数等基本类型，规则是宽松的：大小和精确类型被忽略，因此任何有符号或无符号的整数值都可以与任何其他整数值进行比较。（比较的是`算术值`，而不是`位模式`，因此所有负整数都小于所有无符号整数）
但是，像往常一样，不能将 `int` 与 `float32` 等进行比较。

```plaintext
eq
	等于，支持2个或更多个参数，效果类似于 arg1==arg2 || arg1==arg3 || arg1==arg4 ...
	区别于 Go语言中的 `||`, 这里的所有参数都会进行取值运算
	Returns the boolean truth of arg1 == arg2
ne
	不等于
	Returns the boolean truth of arg1 != arg2
lt
	小于
	Returns the boolean truth of arg1 < arg2
le
	小于等于
	Returns the boolean truth of arg1 <= arg2
gt
	大于
	Returns the boolean truth of arg1 > arg2
ge
	大于等于
	Returns the boolean truth of arg1 >= arg2
````

### 添加模板函数

```golang

// Define a simple template to test the functions.
const tmpl = `{{ . | lower | repeat }}`

// Define the template funcMap with two functions.
var funcMap = template.FuncMap{
	"lower":  strings.ToLower,
	"repeat": func(s string) string { return strings.Repeat(s, 2) },
}

// Define a New template, add the funcMap using Funcs and then Parse
// the template.
parsedTmpl, err := template.New("t").Funcs(funcMap).Parse(tmpl)
if err != nil {
	log.Fatal(err)
}

// [Funcs] must be called before a template is parsed to add
// functions to the template. [Funcs] can also be used after a
// template is parsed to overwrite template functions.
//
// Here the function identified by "repeat" is overwritten.
parsedTmpl.Funcs(template.FuncMap{
	"repeat": func(s string) string { return strings.Repeat(s, 3) },
})

```

## 嵌套模板定义

在解析模板时，可以定义另一个模板并将其与正在解析的模板相关联。模板定义必须出现在模板的顶层，就像 Go 程序中的全局变量一样。  
用 `define` 和 `end` 操作围绕每个模板声明。

```tpl
{{define "T1"}}ONE{{end}}
{{define "T2"}}TWO{{end}}
{{define "T3"}}{{template "T1"}} {{template "T2"}}{{end}}
{{template "T3"}}
```

如果执行，此模板将生成文本

```bash
ONE TWO
```

## 关联模板

每个模板都由创建时指定的字符串命名。此外，每个模板都与零个或多个其他模板相关联，它可以按名称调用这些模板；这种关联是可传递的，并形成模板的名称空间。
模板可以使用模板调用来实例化另一个关联的模板；名称必须是与包含调用的模板关联的模板的名称。
可以多次调用 `Parse` 来组装各种相关模板，也可以使用 `ParseFiles` 或 `ParseGlob` 简单地解析存储在文件中的相关模板。

```golang
// pattern is the glob pattern used to find all the template files.
pattern := filepath.Join(dir, "*.tmpl")

// Here starts the example proper.
// T0.tmpl is the first name matched, so it becomes the starting template,
// the value returned by ParseGlob.
tmpl := template.Must(template.ParseGlob(pattern))
```

## 模板渲染/执行

模板可以通过以下两种方法执行/渲染：

```golang
err := tmpl.Execute(os.Stdout, "no data needed")
if err != nil {
	log.Fatalf("execution failed: %s", err)
}

```

```golang
err := tmpl.ExecuteTemplate(os.Stdout, "T2", "no data needed")
if err != nil {
	log.Fatalf("execution failed: %s", err)
}
```

## 修改默认的分隔符

模板引擎默认使用 `{{` 和 `}}` 作为分隔符，但这可能和一些前端框架冲突，这个时候我们可能需要修改分隔符：

```golang
template.New("test").Delims("{[", "]}").ParseFiles("./t.tmpl")
```

## 参考资料

[text/template](https://pkg.go.dev/text/template)  
[html/template](https://pkg.go.dev/html/template)
