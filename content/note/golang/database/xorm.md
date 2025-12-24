---
date: "2025-12-24T20:43:05+08:00"
title: "XORM -- 入门"
tags: ["Golang", "XORM", "Database"]
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

## 简介

`xorm` 是 Go 语言中一款轻量、高性能、易扩展的 `ORM（Object Relational Mapping，对象关系映射）`库。

<!--more-->

核心特性：

1. 极简的 CRUD 操作：一行代码实现单表的增删改查，无需拼接 SQL；
2. 结构体与表自动映射：通过结构体标签（如 xorm:"pk autoincr"）定义表结构，支持自动创建 / 迁移表；
3. 链式查询：类似 SQL 语法的链式调用（如 Where()、Limit()、Join()），可读性高；
4. 高性能：底层优化了 SQL 执行和数据映射，支持连接池、批量操作，性能接近原生 SQL；
5. 灵活扩展：支持原生 SQL 嵌入、事务、钩子（Hook）、读写分离、主从复制等；
6. 工具链完善：提供 xorm reverse 工具，可从数据库表反向生成 Go 结构体，提升开发效率。

## 安装

```bash
# 安装xorm
go get xorm.io/xorm
# 安装MySQL数据库驱动
go get github.com/go-sql-driver/mysql
```

`Xorm` 支持一下数据库:

| 数据库          | 驱动代码库                       |
| :-------------- | :------------------------------- |
| Mysql           | github.com/go-sql-driver/mysql   |
| MyMysql         | github.com/ziutek/mymysql/godrv  |
| Tidb            | github.com/pingcap/tidb          |
| Postgres        | github.com/lib/pq                |
| Postgres        | github.com/jackc/pgx             |
| SQLite          | github.com/mattn/go-sqlite3      |
| SQLite(Pure Go) | modernc.org/sqlite               |
| MsSql           | github.com/denisenkom/go-mssqldb |
| Dameng          | gitee.com/travelliu/dm           |
| Oracle          | github.com/godror/godror         |
| Oracle          | github.com/mattn/go-oci8         |

## CRUD

### 创建 ORM 引擎

`Xorm` 中所有操作都需要先创建 ORM 引擎（Engine 引擎和 Engine Group 引擎）。  
一个 Engine 引擎用于对单个数据库进行操作，一个 Engine Group 引擎用于对读写分离的数据库或者负载均衡的数据库进行操作。  
Engine 引擎和 EngineGroup 引擎的 API 基本相同，所有适用于 Engine 的 API 基本上都适用于 EngineGroup，并且可以比较容易的从 Engine 引擎迁移到 EngineGroup 引擎。

本文以为介绍 `Xorm` 基础操作为目的，故仅对提供基础的 Engine 创建：示例

```golang
import (
    _ "github.com/go-sql-driver/mysql"
    "xorm.io/xorm"
)

var engine *xorm.Engine

func main() {
    var err error
    engine, err = xorm.NewEngine("mysql", "root:123@/test?charset=utf8")
    ...
}
```

<!-- TODO 代码 效果 -->
### 建表

### 插入

<!-- 如果不存在则插入，否则更新 修改指定行。。。 -->

### 查询

<!-- 条件查询 查询指定列 -->

### 更新

<!-- 更新全部  更新指定列 -->

### 删除

<!-- 软删除 硬删除 -->

## 事务处理

<!-- 简单的加减事务 -->

<!-- ## Reverse 工具 单独开章节-->

<!-- ## 缓存 -->

## 参考资料

[Xorm 官方文档](https://xorm.io/zh/docs/chapter-01/1.engine/)
