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

##

## 参考资料

[text/template](https://pkg.go.dev/text/template)  
[html/template](https://pkg.go.dev/html/template)
