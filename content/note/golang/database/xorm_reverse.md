---
date: "2026-01-13T12:11:16+08:00"
title: "Xorm -- 数据库反转工具 reverse"
tags: ["Golang", "XORM", "ORM", "Database"]
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

`reverse` 是一个用于进行数据库反转的工具，它可以将已有的数据库反转成代码，可以避免人工反转数据库的繁琐、以及由此产生的错误。

<!--more-->

## 安装

> [!important] `xorm` 工程依赖于 `CGO`，请注意要安装 `CGO` 环境。

```bash
go get xorm.io/reverse
```

## 使用

`xorm` 的反转以来一个配置文件，我们通过 `-f` 参数指定使用的配置文件。

```bash
reverse -f example/custom.yml
```

## 配置文件

最简单的配置文件示例：

```yml
kind: reverse
name: mydb
source:
  database: sqlite3
  conn_str: "../testdata/test.db"
targets:
  - type: codes
    language: golang
    output_dir: ../models
```

除了使用默认配置反转代码，我们通过编写 `template` 来控制反转代码输出：

```yml
kind: reverse
name: mydb
source:
  database: sqlite
  conn_str: ../testdata/test.db
targets:
  - type: codes
    include_tables: # 包含的表，以下可以用 **
      - a
      - b
    exclude_tables: # 排除的表，以下可以用 **
      - c
    table_mapper: snake # 表名到代码类或结构体的映射关系
    column_mapper: snake # 字段名到代码或结构体成员的映射关系
    table_prefix: "" # 表前缀
    multiple_files: true # 是否生成多个文件
    language: golang
    template: | # 生成模板，如果这里定义了，优先级比 template_path 高
      package models

      {{$ilen := len .Imports}}
      {{if gt $ilen 0}}
      import (
        {{range .Imports}}"{{.}}"{{end}}
      )
      {{end}}

      {{range .Tables}}
      type {{TableMapper .Name}} struct {
      {{$table := .}}
      {{range .ColumnsSeq}}{{$col := $table.GetColumn .}}	{{ColumnMapper $col.Name}}	{{Type $col}} `{{Tag $table $col}}`
      {{end}}
      }
      {{end}}
    template_path: ./template/goxorm.tmpl # 生成的模板的路径，优先级比 template 低，但比 language 中的默认模板高
    output_dir: ./models # 代码生成目录
```

配置项说明：

- `source`： 配置反转所使用的数据库信息
  - `database`：指定数据库类型，目前支持 `mysql`,`mssqldb`,`pg`,`sqlite3`
  - `conn_str`：数据库连接信息，与 `xorm.NewEngine` 中使用的格式相同
- `targets`：生成代码配置，支持生成多个
  - `type`：默认填写 `codes`，事实上并没有使用
  - `include_tables`：需要反转的表，如果不填写则默认反转所有表。
  - `exclude_tables`：需要排除的表
  - `table_mapper`、`column_mapper`：表名、列名规则，支持 `sanke(默认)`,`gonic`,`same`
    - `sanke`：基于大写字母拆分添加 `_` 后将所有字母小写，如：`UserName -> user_name`,`UserID --> user_i_d`
    - `gonic`: 在 `sanke` 的基础上进行了增强，内置了高频缩写库，如：`UserName -> user_name`,`UserID --> user_id`
    - `same`：保持和数据库中一样
  - `table_prefix`：告知 `reverse` 数据库表名的前缀，在反转前会尝试去除前缀
  - `multiple_files`：是否生成多个文件，如果是则为每一个表生成一个反转文件
  - `language`：反转语言，目前经支持 `Go` 语言（有内置配置）
  - `template`：自定义模板，优先级最高
  - `template_path`：模板文件路径，优先级仅次于 `template`
  - `output_dir`：反转代码输出目录

## 模板语法

### 模板函数

- `UnTitle`: 将单词的第一个字母小写。
- `Upper`: 将单词转为全部大写。
- `TableMapper`: 将表名转为结构体名的映射函数。
- `ColumnMapper`: 将字段名转为结构体成员名的函数。

### Go 语言模版函数

- `Type`: 返回 Go 语言的类型
- `Tag`: 返回 Go 语言的 Tag 信息

### 模板变量

- `Tables`: 所有表。
- `Imports`: 所有需要的导入。对于 `Go` 语言，当反转类型中包含 `time.Time` 时，为 "time"
  对于 `Go` 语言在模板头部添加一下代码段即可：
  ```go-template
  {{$ilen := len .Imports}}
  {{if gt $ilen 0}}
  import (
  {{range .Imports}}"{{.}}"{{end}}
  )
  {{end}}
  ```

### 实用模板块示例

- 反转数据库访问模型 `model`

  ```go-template
    package models

    {{$ilen := len .Imports}}
    {{if gt $ilen 0}}
    import (
    {{range .Imports}}"{{.}}"{{end}}
    )
    {{end}}

    {{range .Tables}}
    type {{TableMapper .Name}} struct {
    {{$table := .}}
    {{range .ColumnsSeq}}{{$col := $table.GetColumn .}}	{{ColumnMapper $col.Name}}	{{Type $col}} `{{Tag $table $col}}`
    {{end}}
    }
    {{end}}
  ```

- 反转数据库表结构同步代码

  > [!important] 需要注意 `xorm.Sync` 会读取数据库所有表信息，所以使用时尽量在一个 `xorm.Sync` 中完成所有同步。

  ```go-template
  func SyncAll(orm *xorm.Engine) error {
      err := orm.Sync(
      {{range .Tables}}&{{TableMapper .Name}}{},
      {{end}}
      )

      return err
  }
  ```

## 参考资料

[Xorm 官方文档](https://xorm.io/)  
[Xorm_reverse 源码](https://gitea.com/xorm/reverse)
