---
date: "2021-03-19T14:37:44+08:00"
title: "GORM -- 一个使用 Go 语言编写的 ORM 框架"
tags: ["Golang", "GORM", "Database", "ORM"]
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

[GORM](https://gorm.io/) 是一个使用 Go 语言编写的 ORM 框架。它文档（多语种）齐全，对开发者友好。支持主流关系型数据库（支持 SQL 语句），如 MySQL、MSSQL、PostgreSQL 等。使用 GORM 只需简单的几个函数调用就可实现对数据库复杂操作，极大地提高开发效率，降低开发门槛。<!-- more -->

## 什么是 ORM？

**Object Relation Mapping（关系对象映射）**

把对象模型表示的对象映射到基于 SQL 的关系模型数据库结构中，在具体的操作实体对象的时候，不需要直接与复杂的 SQL 语句打交道，只需简单的操作实体对象的属性和方法，对于开发者更加友好，不必学习 SQL 语句就可以方便地进行数据库的使用。对于一些中小型项目，使用 ORM 可以极大地提高开发效率，降低开发门槛。另外，也能在一定程度上防止 SQL 注入。

相应的也要以牺牲执行性能、牺牲灵活性、弱化 SQL 能力作为代价。对于高性能开发，并不推荐使用。

## GORM 特性

- 全功能 ORM
- 关联（Has One，Has Many，Belongs To，Many To Many，多态，单表继承）
- Create，Save，Update，Delete，Find 中钩子方法
- 支持 Preload、Joins 的预加载
- 事务，嵌套事务，Save Point，Rollback To Saved Point
- Context、预编译模式、DryRun 模式
- 批量插入，FindInBatches，Find/Create with Map，使用 SQL 表达式、Context Valuer 进行 CRUD
- SQL 构建器，Upsert，数据库锁，Optimizer/Index/Comment Hint，命名参数，子查询
- 复合主键，索引，约束
- Auto Migration
- 自定义 Logger
- 灵活的可扩展插件 API：Database Resolver（多数据库，读写分离）、Prometheus...

## 简单入门（MySQL 版）

### 包准备

```sh
go get -u gorm.io/gorm
go get gorm.io/driver/mysql
```

### 代码

```go
package main

import (
	"fmt"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

// 连接 MySQL 所需信息
var (
	// 用户
	user string = "root"
	// 密码
	pwd string = "123456"
	// IP 地址，带端口号
	addr string = "localhost:3306"
	// 数据库名
	dbName string = "chart_room"
)

type Product struct {
	// 官方提供的 Model 作为匿名字段，包含 ID、CreatedAt、UpdatedAt、DeletedAt 四个字段
	// 其中 ID 为默认主键，DeletedAt 默认添加索引
	gorm.Model
	Code  string
	Price uint
}

func main() {
	// parseTime 查询结果是否自动解析为时间，Loc 为时区设置
	dsn := fmt.Sprintf("%s:%s@tcp(%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		user, pwd, addr, dbName)
	// 连接数据库
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		// 连接失败
		panic("failed to connect database")
	}

	// 迁移 schema
	// 执行后数据库会自动创建相应的表
	// 表名为小写结构体名加 s，表信息可以在后面查看
	// 有两个单词时，默认用下划线连接
	db.AutoMigrate(&Product{})

	// Create
	// 向数据库中添加一条记录
	db.Create(&Product{Code: "D42", Price: 100})

	// Read
	var product Product
	db.First(&product, 1)                      // 根据整型主键查找，查找结果保存在 product 中
	db.First(&product, "code = ?", "D42")       // 查找 code 字段值为 D42 的记录

	// Update - 将 product 的 price 更新为 200
	db.Model(&product).Update("Price", 200)
	// Update - 更新多个字段
	db.Model(&product).Updates(Product{Price: 200, Code: "F42"}) // 仅更新非零值字段
	db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})

	// Delete - 删除 product，根据主键
	db.Delete(&product, 1)
}
```

## GORM 模型

```go
// Model 可以直接作为字段添加到我们的结构体中，加速开发
type Model struct {
	// ID 默认为主键
	ID        uint `gorm:"primarykey"`
	CreatedAt time.Time
	UpdatedAt time.Time
	// DeletedAt 默认创建索引，允许重复
	DeletedAt DeletedAt `gorm:"index"`
}
```

## 创建表信息

```sh
mysql> desc products;
+------------+-----------------+------+-----+---------+----------------+
| Field      | Type            | Null | Key | Default | Extra          |
+------------+-----------------+------+-----+---------+----------------+
| id         | bigint unsigned | NO   | PRI | NULL    | auto_increment |
| created_at | datetime(3)     | YES  |     | NULL    |                |
| updated_at | datetime(3)     | YES  |     | NULL    |                |
| deleted_at | datetime(3)     | YES  | MUL | NULL    |                |
| code       | longtext        | YES  |     | NULL    |                |
| price      | bigint unsigned | YES  |     | NULL    |                |
+------------+-----------------+------+-----+---------+----------------+
```

## 常用方法

```go
// 自定义表名创建表
db.Table(tableName).CreateTable(&Product{})

// 创建记录并更新给出的字段。
db.Select("Name", "Age", "CreatedAt").Create(&user)

// 根据 ID 更新，即使是零值也会修改
db.Save(&user)

// Email 的 ID 是 `10`
db.Delete(&email)
// DELETE from emails where id = 10;

// 带额外条件的删除
db.Where("name = ?", "jinzhu").Delete(&email)
// DELETE from emails where id = 10 AND name = "jinzhu";

// 根据主键删除数据
db.Delete(&User{}, 10)
// DELETE FROM users WHERE id = 10;

db.Delete(&User{}, "10")
// DELETE FROM users WHERE id = 10;

db.Delete(&users, []int{1, 2, 3})
// DELETE FROM users WHERE id IN (1,2,3);

// 查询软删除的数据
db.Unscoped().Where("age = 20").Find(&users)
// SELECT * FROM users WHERE age = 20;

// 永久删除
db.Unscoped().Delete(&order)
// DELETE FROM orders WHERE id = 10;
```

## 查询条件

### Where

```go
// 基本条件
db.Where("name = ?", "张三").Find(&users)
// SELECT * FROM users WHERE name = '张三';

// 多个条件
db.Where("name = ? AND age >= ?", "张三", 18).Find(&users)
// SELECT * FROM users WHERE name = '张三' AND age >= 18;

// IN 查询
db.Where("name IN ?", []string{"张三", "李四"}).Find(&users)
// SELECT * FROM users WHERE name IN ('张三','李四');

// Like 模糊查询
db.Where("name LIKE ?", "%张%").Find(&users)
// SELECT * FROM users WHERE name LIKE '%张%';

// 结构体条件（仅非零字段生效）
db.Where(&User{Name: "张三", Age: 0}).Find(&users)
// SELECT * FROM users WHERE name = '张三';

// Map 条件
db.Where(map[string]interface{}{"name": "张三", "age": 18}).Find(&users)
// SELECT * FROM users WHERE name = '张三' AND age = 18;
```

### Not

```go
db.Not("name = ?", "张三").Find(&users)
// SELECT * FROM users WHERE NOT name = '张三';

db.Not("name IN ?", []string{"张三", "李四"}).Find(&users)
// SELECT * FROM users WHERE name NOT IN ('张三','李四');
```

### Or

```go
db.Where("age >= ?", 18).Or("name = ?", "张三").Find(&users)
// SELECT * FROM users WHERE age >= 18 OR name = '张三';
```

### 链式条件

```go
db.Where("age >= ?", 18).Where("name LIKE ?", "%张%").Find(&users)
// SELECT * FROM users WHERE age >= 18 AND name LIKE '%张%';

db.Where("age >= ?", 18).Or("name = ?", "张三").Where("active = ?", true).Find(&users)
// SELECT * FROM users WHERE (age >= 18 OR name = '张三') AND active = true;
```

## 排序与分页

```go
// 排序
db.Order("age desc").Find(&users)
// SELECT * FROM users ORDER BY age desc;

db.Order("age desc").Order("name asc").Find(&users)
// SELECT * FROM users ORDER BY age desc, name asc;

// 分页
db.Offset(10).Limit(5).Find(&users)
// SELECT * FROM users OFFSET 10 LIMIT 5;

// 常见分页模式
page := 1
pageSize := 20
db.Offset((page - 1) * pageSize).Limit(pageSize).Find(&users)
```

## 聚合查询

```go
// Count
var count int64
db.Model(&User{}).Where("age >= ?", 18).Count(&count)
// SELECT count(*) FROM users WHERE age >= 18;

// Sum
var totalAge int64
db.Model(&User{}).Select("sum(age)").Scan(&totalAge)

// Group + Having
db.Select("age, count(*) as count").Group("age").Having("count(*) > 5").Find(&result)
// SELECT age, count(*) as count FROM users GROUP BY age HAVING count(*) > 5;
```

## 关联查询

### Preload（预加载关联）

```go
type User struct {
	gorm.Model
	Name  string
	Books []Book
}

type Book struct {
	gorm.Model
	Title  string
	UserID uint
}

// 查询用户时预加载关联的 Books
var user User
db.Preload("Books").First(&user, 1)
// SELECT * FROM users WHERE id = 1;
// SELECT * FROM books WHERE user_id = 1;

// 预加载所有关联
db.Preload(clause.Associations).First(&user, 1)

// 嵌套预加载
db.Preload("Books.Chapters").First(&user, 1)
```

### Joins（联表查询）

```go
// 基本 Joins
db.Joins("JOIN books ON books.user_id = users.id").Find(&users)
// SELECT * FROM users JOIN books ON books.user_id = users.id;

// 带条件的 Joins
db.Joins("JOIN books ON books.user_id = users.id AND books.active = ?", true).Find(&users)

// Joins + Preload
db.Joins("JOIN books ON books.user_id = users.id").Preload("Books").Find(&users)
```

## 事务

```go
// 使用 Transaction 方法（推荐）
err := db.Transaction(func(tx *gorm.DB) error {
	// 在事务中创建用户
	if err := tx.Create(&user).Error; err != nil {
		return err // 返回错误，事务回滚
	}
	// 在事务中更新账户
	if err := tx.Model(&account).Update("balance", gorm.Expr("balance - ?", 100)).Error; err != nil {
		return err // 返回错误，事务回滚
	}
	return nil // 返回 nil，事务提交
})

// 手动事务控制
tx := db.Begin()
defer func() {
	if r := recover(); r != nil {
		tx.Rollback()
	}
}()

if err := tx.Create(&user).Error; err != nil {
	tx.Rollback()
	return err
}

if err := tx.Model(&account).Update("balance", gorm.Expr("balance - ?", 100)).Error; err != nil {
	tx.Rollback()
	return err
}

tx.Commit()
```

## 常用的结构体标记（Tag）

| 结构体标记（Tag） | 描述                                                     |
| :---------------- | :------------------------------------------------------- |
| Column            | 指定列名                                                 |
| Type              | 指定列数据类型                                           |
| Size              | 指定列大小，默认值 255                                   |
| PRIMARY_KEY       | 将列指定为主键                                           |
| UNIQUE            | 将列指定为唯一                                           |
| DEFAULT           | 指定列默认值                                             |
| PRECISION         | 指定列精度                                               |
| NOT NULL          | 将列指定为非 NULL                                        |
| AUTO_INCREMENT    | 指定列是否为自增类型                                     |
| INDEX             | 创建具有或不带名称的索引，如果多个索引同名则创建复合索引 |
| UNIQUE_INDEX      | 和 INDEX 类似，只不过创建的是唯一索引                    |
| EMBEDDED          | 将结构设置为嵌入                                         |
| EMBEDDED_PREFIX   | 设置嵌入结构的前缀                                       |
| -                 | 忽略此字段                                               |
