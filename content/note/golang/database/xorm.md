---
date: "2025-12-24T20:43:05+08:00"
title: "XORM -- 简单入门"
tags: ["Golang", "XORM", "Database"]
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
    engine, err = xorm.NewEngine("mysql", "root:root123@/test?charset=utf8")
    ...
}
```

### 建表

```golang
package main

import (
	"log"
	"time"

	_ "github.com/go-sql-driver/mysql"
	"xorm.io/xorm"
)

type User struct {
	Id      int64
	Name    string
	Salt    string
	Age     int
	Passwd  string    `xorm:"varchar(200)"`
	Created time.Time `xorm:"created"`
	Updated time.Time `xorm:"updated"`
}

func main() {
	engine, err := xorm.NewEngine("mysql", "root:root123@/test?charset=utf8")
	if err != nil {
		log.Println(err)
		return
	}
	err = engine.Sync2(new(User))
	if err != nil {
		log.Println(err)
		return
	}

	log.Println("同步数据库成功")
}


```

厨师状态下数据库 `test` 是没有表的，通过 `Engin` 的 `Sync2` 方法(`Sync`也行)，将 `User` 对象映射为表结构，便在数据库中完成同步。 可以通过 `show tables` 和 `describe user` 查看表结构同比情况。

```bash

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| user           |
+----------------+
1 row in set (0.00 sec)

mysql> describe user;
+---------+--------------+------+-----+---------+----------------+
| Field   | Type         | Null | Key | Default | Extra          |
+---------+--------------+------+-----+---------+----------------+
| id      | bigint(20)   | NO   | PRI | NULL    | auto_increment |
| name    | varchar(255) | YES  |     | NULL    |                |
| salt    | varchar(255) | YES  |     | NULL    |                |
| age     | int(11)      | YES  |     | NULL    |                |
| passwd  | varchar(200) | YES  |     | NULL    |                |
| created | datetime     | YES  |     | NULL    |                |
| updated | datetime     | YES  |     | NULL    |                |
+---------+--------------+------+-----+---------+----------------+
7 rows in set (0.00 sec)

```

可以一看到， `test` 数据库中创建了一张 `user` 表，其中 `id` 与 `User` 中的 `ID` 字段对应，被用作了自增主键。其他字段也更具数据类型进行了相应映射。

### 插入

#### 插入单条数据

插入一条数据，此时可以用 `Insert` 或者 `InsertOne` 。

```golang
    user := &User{
        Id:     0,
        Name:   "tom",
        Salt:   "salt",
        Age:    18,
        Passwd: "123456",
    }
    aff, err := engine.Insert(user)
    if err != nil {
        log.Println(err)
        return
    }
    fmt.Println(aff, user) // 1 &{1 tom salt 18 123456 2025-12-26 21:05:43.840153755 +0800 CST 2025-12-26 21:05:43.840213981 +0800 CST}
```

在插入单条数据成功后，如果该结构体有自增字段(设置为 autoincr)，则自增字段会被自动赋值为数据库中的 id。这里需要注意的是，如果插入的结构体中，自增字段已经赋值，则该字段会被作为非自增字段插入。
查询数据库 `user` 表中的数据，可以看到插入的数据如下：

```bash
select * from user;
+----+------+------+------+--------+---------------------+---------------------+
| id | name | salt | age  | passwd | created             | updated             |
+----+------+------+------+--------+---------------------+---------------------+
|  1 | tom  | salt |   18 | 123456 | 2025-12-26 21:05:43 | 2025-12-26 21:05:43 |
+----+------+------+------+--------+---------------------+---------------------+
1 row in set (0.00 sec)
```

#### 插入多条数据

```golang
// 插入同一个表的多条数据，此时如果数据库支持批量插入，那么会进行批量插入，但是这样每条记录就无法被自动赋予id值。如果数据库不支持批量插入，那么就会一条一条插入
users := make([]User, 1)
users[0].Name = "name0"
...
affected, err := engine.Insert(&users)
```

```golang
// 使用指针Slice插入多条记录，同上
users := make([]*User, 1)
users[0] = new(User)
users[0].Name = "name0"
...
affected, err := engine.Insert(&users)
```

### 查询

#### Get 方法

> `ShowSQL(true)` 使 xorm 打印出具体执行的 SQL 语句, 在学习 xorm 和排查问题时候很有用。

```golang
engine.ShowSQL(true)

user1 := new(User)
fmt.Println(engine.ID(1).Get(user1))
fmt.Println(user1)

user2 := &User{
    Id: 1,
}
fmt.Println(engine.Get(user2))
fmt.Println(user2)

user3 := new(User)
fmt.Println(engine.Where("id = ?", 1).Get(user3))
fmt.Println(user3)
```

上面使用三种方法利用 `Get` 查询数据，执行结果如下：

```bash
[xorm] [info]  2025/12/26 21:22:53.448737 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1 [1] - 1.059202ms
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
[xorm] [info]  2025/12/26 21:22:53.448921 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1 [1] - 78.343µs
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
[xorm] [info]  2025/12/26 21:22:53.449003 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE (id = ?) LIMIT 1 [1] - 59.438µs
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
```

#### Find 方法

区别于 `Get` 方法查询单条数据，`Find` 用于查询多条数据，可以搭配各种条件、分页等使用，也可以单独使用查询全表。

```golang
users := make([]Use, 0)
err := engine.Where("age > ? or name = ?", 30, "xlw").Limit(20, 10).Find(&users)
```

#### Count 方法

统计数据使用 `Count` 方法， `Count` 方法的参数为 `struct` 的指针并且成为查询条件。

```golang
user := new(User)
total, err := engine.Where("id >?", 1).Count(user)
```

#### Exist 方法

`Exist` 方法用来判断某个记录是否存在可以, 相比 `Get` 性能更好。

```golang
has, err = engine.Where("name = ?", "test1").Exist(&RecordExist{})
// SELECT * FROM record_exist WHERE name = ? LIMIT 1
```

### 更新

`Update` 方法将返回两个参数，第一个为 更新的记录数，需要注意的是 `SQLITE` 数据库返回的是根据更新条件查询的记录数而不是真正受更新的记录数。

```golang
user := new(User)
user.Name = "myname"
affected, err := engine.ID(id).Update(user)

// 通过添加Cols函数指定需要更新结构体中的哪些值，未指定的将不更新，指定了的即使为0也会更新
affected, err := engine.ID(id).Cols("age").Update(&user)

// 有时候希望能够指定必须更新某些字段（ 如 改为类型的零值），而其它字段根据值的情况自动判断，可以使用 MustCols 来组合 Update 使用。
affected, err := engine.ID(id).MustCols("age").Update(&user)

```

### 删除

删除数据用 `Delete` 方法，参数为 `struct` 的指针并且成为查询条件。`Delete` 会通过返回值告知删除的记录条数。

```golang
user := new(User)
affected, err := engine.ID(id).Delete(user)
```

> 注意：
> 1：当删除时，如果 user 中包含有 bool,float64 或者 float32 类型，有可能会使删除失败。具体请查看 FAQ  
> 2：必须至少包含一个条件才能够进行删除，这意味着直接用

```golang
engine.Delete(new(User))
// 将会报一个保护性的错误，如果你真的希望将整个表删除，你可以添加一个 Where
engine.Where("1=1").Delete(new(User))
```

#### 软删除

`Deleted` 可以让您不真正的删除数据，而是标记一个删除时间。使用此特性需要在 `xorm` 标记中使用 `deleted` 标记，如下所示进行标记，对应的字段可以为 `time.Time`, `type MyTime time.Time`，`int` 或者 `int64`类型。

```golang
type User struct {
    Id int64
    Name string
    DeletedAt time.Time `xorm:"deleted"`
}

```

当给 `DeletedAt` 加上 `deleted` 的 `Tag` 后,执行的 SQL 会发生相应变化,以实现数据软删除的效果

```golang
var user User
engine.ID(1).Get(&user)
// SELECT * FROM user WHERE id = ?
engine.ID(1).Delete(&user)
// UPDATE user SET ..., deleted_at = ? WHERE id = ?
engine.ID(1).Get(&user)
// 再次调用Get，此时将返回false, nil，即记录不存在
engine.ID(1).Delete(&user)
// 再次调用删除会返回0, nil，即记录不存在
```

此时如果想要 查询实际数据 或者 真正删除数据,需要使用 `Unscoped`:

```golang
var user User
engine.ID(1).Unscoped().Get(&user)
// 此时将可以获得记录
engine.ID(1).Unscoped().Delete(&user)
// 此时将可以真正的删除记录
```

## 事务处理

当使用事务处理时，需要创建 `Session` 对象。在进行事务处理时，可以混用 ORM 方法和 RAW 方法，如下代码所示：

```golang
func MyTransactionOps() error {
    session := engine.NewSession()
    defer session.Close()

    // add Begin() before any action
    if err := session.Begin(); err != nil {
        return err
    }

    user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
    if _, err := session.Insert(&user1); err != nil {
        return err
    }
    user2 := Userinfo{Username: "yyy"}
    if _, err = session.Where("id = ?", 2).Update(&user2); err != nil {
        return err
    }

    if _, err = session.Exec("delete from userinfo where username = ?", user2.Username); err != nil {
        return err
    }

    // add Commit() after all actions
    return session.Commit()
}

```

当然也可以使用封装程度更高的 `Transaction` 方法简化编码:

```golang
res, err := engine.Transaction(func(session *xorm.Session) (interface{}, error) {
    user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
    if _, err := session.Insert(&user1); err != nil {
        return nil, err
    }

    user2 := Userinfo{Username: "yyy"}
    if _, err := session.Where("id = ?", 2).Update(&user2); err != nil {
        return nil, err
    }

    if _, err := session.Exec("delete from userinfo where username = ?", user2.Username); err != nil {
        return nil, err
    }
    return nil, nil
})
```

## 参考资料

[Xorm 官方文档](https://xorm.io/zh/docs/chapter-01/1.engine/)  
[Xorm API 文档](https://pkg.go.dev/xorm.io/xorm/)
