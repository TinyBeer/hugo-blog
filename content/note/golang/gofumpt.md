---
date: "2024-05-24T13:08:28+08:00"
title: "gofumpt -- 更严格的代码格式化策略"
tags: ["Golang", "代码格式化"]
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

[`gofumpt`](https://github.com/mvdan/gofumpt) 比 `gofmt` 更严格的代码格式化策略。它是 `gofmt` 的超集，同时向后兼容。<!-- more -->

> 该工具是 Go 1.22 版本的 gofmt 的一个分支，需要 Go 1.21 或更高版本。
> Vendor 和 testdata 目录将被跳过，除非作为显式参数给出。生成的 Go 文件也是如此。

# 使用

```sh
# 安装
go install mvdan.cc/gofumpt@latest
```

## 常用参数

| 参数 | 说明 |
|------|------|
| `-l` | 列出需要格式化的文件（不修改） |
| `-w` | 就地格式化文件 |
| `-d` | 显示格式化差异（不修改文件） |
| `-extra` | 启用额外的格式化规则 |
| `-lang` | 指定 Go 语言版本（如 `-lang=1.21`），影响部分规则的行为 |

```sh
# 查看哪些文件需要格式化
gofumpt -l .

# 查看格式化差异
gofumpt -d .

# 就地格式化所有 Go 文件
gofumpt -w .

# 启用额外规则
gofumpt -extra -w .
```

## CI/CD 集成

在 CI 中使用 `gofumpt -l .` 检查代码格式。如果存在未格式化的文件，`gofumpt` 会输出文件列表并以非零退出码退出。

```yaml
# GitHub Actions 示例
- name: Check formatting
  run: |
    go install mvdan.cc/gofumpt@latest
    test -z "$(gofumpt -l .)"
```

```yaml
# GitLab CI 示例
format_check:
  script:
    - go install mvdan.cc/gofumpt@latest
    - test -z "$(gofumpt -l .)"
```

## VS Code 配置

`gopls`使用`gofumpt`

```json
"go.useLanguageServer": true,
"gopls": {
	"formatting.gofumpt": true,
},
```

## GoLand 配置

- 打开 Settings (File > Settings)
- 打开 Tools 项
- 找到 File Watchers 子项
- 单击右侧的`+`来添加一个新的文件监视器
- 选择自定义模板 在新窗口中继续操作
- File Types:选择`all .go files`
- Scope: Project Files
- Program:选择 `gofumpt` 可执行文件
- Arguments:`-w $FilePath$`
- Output path to refresh: `$FilePath$`
- Working directory: `$ProjectFileDir$`
- 环境变量:`GOROOT=$GOROOT$;GOPATH=$GOPATH$;PATH=$GoBinDirs$`

为了避免不必要的运行，您应该禁用"高级"部分中的所有复选框。

## Vim / Neovim 配置

### vim-go

在 `.vimrc` 或 `init.vim` 中设置：

```vim
let g:go_fmt_command = "gopls"
let g:go_gopls_gofumpt = 1
```

### govim

```vim
call govim#config#Set("Gofumpt", 1)
```

### Neovim (lspconfig)

使用 [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) 时：

```lua
require('lspconfig').gopls.setup({
    settings = {
        gopls = {
            gofumpt = true
        }
    }
})
```

### Neovim (conform.nvim)

使用 [conform.nvim](https://github.com/stevearc/conform.nvim) 直接调用格式化：

```lua
require("conform").setup({
  formatters_by_ft = {
    go = { "gofumpt" },
  },
})
```

### Emacs

lsp-mode 8.0.0+：

```elisp
(setq lsp-go-use-gofumpt t)
```

lsp-mode < 8.0.0：

```elisp
(lsp-register-custom-settings
 '(("gopls.gofumpt" t)))
```

eglot：

```elisp
(setq-default eglot-workspace-configuration
 '((:gopls . ((gofumpt . t)))))
```

### Helix

编辑 `~/.config/helix/languages.toml`：

```toml
[language-server.gopls.config]
"formatting.gofumpt" = true
```

### Zed

编辑 `settings.json`：

```json
"lsp": {
  "gopls": {
    "initialization_options": {
      "gofumpt": true
    }
  }
}
```

# 新增的规则

> 以下规则为 `gofumpt` 相对于 `gofmt` 新增的格式化策略。使用 `-extra` 标志可启用额外规则（文末标注）。

- 在赋值操作后不换行

  ```go
  func foo() {
  	foo :=
  		"bar"
  }
  ```

  ```go
  func foo() {
  	foo := "bar"
  }
  ```

- 函数体上下不换行

  ```go
  func foo() {

  	println("bar")

  }
  ```

  ```go
  func foo() {
  	println("bar")
  }
  ```

- 多行函数参数列表应使用尾逗号，使 `)` 和 `{` 分开以提高可读性

  ```go
  func foo(s string,
  	i int) {
  	println("bar")
  }

  // With an empty line it's slightly better, but still not great.
  func bar(s string,
  	i int) {

  	println("bar")
  }
  ```

  ```go
  func foo(s string,
  	i int,
  ) {
  	println("bar")
  }

  // With an empty line it's slightly better, but still not great.
  func bar(s string,
  	i int,
  ) {
  	println("bar")
  }
  ```

- 代码块中单条语句或注释不空行

  ```go
  if err != nil {

  	return err
  }
  ```

  ```go
  if err != nil {
  	return err
  }
  ```

- 在简单的错误检查之前没有空行

  ```go
  foo, err := processFoo()

  if err != nil {
  	return err
  }
  ```

  ```go
  foo, err := processFoo()
  if err != nil {
  	return err
  }
  ```

- 复合字面量应该一致地使用换行符

  ```go
  // A newline before or after an element requires newlines for the opening and
  // closing braces.
  var ints = []int{1, 2,
  	3, 4}

  // A newline between consecutive elements requires a newline between all
  // elements.
  var matrix = [][]int{
  	{1},
  	{2}, {
  		3,
  	},
  }
  ```

  ```go
  var ints = []int{
  	1, 2,
  	3, 4,
  }

  var matrix = [][]int{
  	{1},
  	{2},
  	{
  		3,
  	},
  }
  ```

- 多行函数调用的闭括号 `)` 应放在新行的开头

  ```go
  result := compute(
  	a,
  	b,
  	c)
  ```

  ```go
  result := compute(
  	a,
  	b,
  	c,
  )
  ```

- 空字段列表应该使用一行

  ```go
  var V interface {
  } = 3

  type T struct {
  }

  func F(
  )
  ```

  ```go
  var V interface{} = 3

  type T struct{}

  func F()
  ```

- 标准库导入应放在顶部单独分组

  ```go
  import (
  	"foo.com/bar"

  	"io"

  	"io/ioutil"
  )
  ```

  ```go
  import (
  	"io"
  	"io/ioutil"

  	"foo.com/bar"
  )
  ```

- 短的 case 判断应该在一行内处理

  ```go
  switch c {
  case 'a', 'b',
  	'c', 'd':
  }
  ```

  ```go
  switch c {
  case 'a', 'b', 'c', 'd':
  }
  ```

- 多行顶级声明必须用空行分隔

  ```go
  func foo() {
  	println("multiline foo")
  }
  func bar() {
  	println("multiline bar")
  }
  ```

  ```go
  func foo() {
  	println("multiline foo")
  }

  func bar() {
  	println("multiline bar")
  }
  ```

- 单条变量声明不应该使用圆括号

  ```go
  var (
  	foo = "bar"
  )
  ```

  ```go
  var foo = "bar"
  ```

- 连续的顶级声明应该组合在一起

  ```go
  var nicer = "x"
  var with = "y"
  var alignment = "z"
  ```

  ```go
  var (
  	nicer     = "x"
  	with      = "y"
  	alignment = "z"
  )
  ```

- 简单的 var 声明语句应该使用短赋值（仅在函数体内生效）

  ```go
  func main() {
  	var s = "somestring"
  }
  ```

  ```go
  func main() {
  	s := "somestring"
  }
  ```

- 默认情况下启用`-s`代码简化标志

  ```go
  var _ = [][]int{[]int{1}}
  ```

  ```go
  var _ = [][]int{{1}}
  ```

- 八进制整数字面量应该在使用 Go 1.13 及更高版本的模块上使用 `0o` 前缀

  ```go
  const perm = 0755
  ```

  ```go
  const perm = 0o755
  ```

- 不是 Go 编译器指令的注释应该以空格开头

  ```go
  //go:noinline

  //Foo is awesome.
  func Foo() {}
  ```

  ```go
  //go:noinline

  // Foo is awesome.
  func Foo() {}
  ```

- 无用的括号应该被移除

  ```go
  type C chan (int)

  var _ = f((3))
  ```

  ```go
  type C chan int

  var _ = f(3)
  ```

  注意：二元/一元表达式周围的括号，以及需要括号的类型（如 `chan (<-chan T)`）会保留。

- 复合字面量不应该有开头或结尾的空行

  ```go
  var _ = []string{

  	"foo",

  }

  var _ = map[string]string{

  	"foo": "bar",

  }
  ```

  ```go
  var _ = []string{
  	"foo",
  }

  var _ = map[string]string{
  	"foo": "bar",
  }
  ```

- 字段列表不应该有开头或结尾的空行

  ```go
  type Person interface {

  	Name() string

  	Age() int

  }

  type ZeroFields struct {

  	// No fields are needed here.

  }
  ```

  ```go
  type Person interface {
  	Name() string

  	Age() int
  }

  type ZeroFields struct {
  	// No fields are needed here.
  }
  ```

- 相邻的相同类型的参数应分组在一起
  这个规则需要使用 `-extra`。该规则对简单参数效果不错，但复杂参数时容易造成误读，按需使用即可。

  ```go
  func Foo(bar string, baz string) {}
  ```

  ```go
  func Foo(bar, baz string) {}
  ```

- 避免使用 naked return，使用显式返回值以提高可读性
  这个规则需要使用 `-extra`。

  ```go
  func Foo() (err error) {
  	return
  }
  ```

  ```go
  func Foo() (err error) {
  	return err
  }
  ```
