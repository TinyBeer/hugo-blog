---
date: "2024-06-09T19:23:11+08:00"
title: "MongoDB -- 学习笔记"
tags: ["Database", "MongoDB"]
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

[MongoDB](https://www.mongodb.com/)是一个基于分布式文件存储的数据库。由 C++语言编写。旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。它支持的数据结构非常松散，是类似 json 的 bson 格式，因此可以存储比较复杂的数据类型。MongoDB 最大的特点是它支持的查询语言非常强大，其语法有点类似于面向对象的查询语言，几乎可以实现类似关系数据库单表查询的绝大部分功能，而且还支持对数据建立索引。

# Docker 安装

```shell
# 文档使用
# 拉取镜像
docker pull mongo:4.2
# 运行mongodb镜像
docker run --name mongo -p 27017:27017 -d mongo:4.2

# dockerhub提供
# 拉取镜像
docker pull mongo
# 运行mongodb镜像
docker run --name some-mongo -d mongo
# 带存储映射
docker run --name mongodb -p 27017:27017 -v $PWD/db:/data/db -d mongo:latest


# 官网提供
# MongoDB 5.0+ Docker 映像需要 AVX 系统支持。如果您的系统不支持 AVX，则可以使用之前版本的 MongoDB docker 映像。
# 拉取最新版镜像
docker pull mongodb/mongodb-community-server:latest
# 启动mongodb
docker run --name mongodb -p 27017:27017 -d mongodb/mongodb-community-server:latest
```

# MongoDB Shell CRUD 操作

## 连接 mongo

```shell
# 进入容器
docker exec -it mongodb mongo admin

# 创建管理员用户
use admin;
db.createUser({ user: 'admin', pwd: 'admin1234', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
# 登录
db.auth("admin","admin1234");
# 创建zero用户
db.createUser({ user: 'zero', pwd: '123456', roles: [ { role: "readWriteAnyDatabase", db: "admin" } ] });
```

> 这里需要注意你创建用户使用的数据库是哪一个。如果在非注册数据库认证，可能会认证失败。使用代码连接时，也需要提供注册时用的数据库。
> 为了避免麻烦，还是把所有用户都注册在 admin 里，方便管理。

## 插入文档

- 如果该集合当前不存在，则插入操作将创建该集合。
- 在 MongoDB 中，存储在集合中的每个文档都需要一个唯一的 \_id 字段作为主键。如果插入的文档省略了 \_id 字段，MongoDB 驱动程序会自动为 ObjectId 字段生成一个 \_id。
- MongoDB 中的所有写入操作在单个文档级别上都是原子操作。
- 对于写关注，您可以指定 MongoDB 请求的写操作确认级别。

### 插入一条数据

```shell
# 插入一条数据
db.inventory.insertOne(
   { item: "canvas", qty: 100, tags: ["cotton"], size: { h: 28, w: 35.5, uom: "cm" } }
)
# 查询inventory中item为"canvas"的对象
db.inventory.find( { item: "canvas" } )
```

### 插入多条数据

```shell
db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } },
   { item: "mat", qty: 85, tags: ["gray"], size: { h: 27.9, w: 35.5, uom: "cm" } },
   { item: "mousepad", qty: 25, tags: ["gel", "blue"], size: { h: 19, w: 22.85, uom: "cm" } }
])

# 查询inventory中所有
db.inventory.find({})
```

### 其他插入方法

- `db.collection.updateOne()` 与 `upsert: true` 选项一起使用时。
- `db.collection.updateMany()` 与 `upsert: true` 选项一起使用时。
- `db.collection.findAndModify()` 与 `upsert: true` 选项一起使用时。
- `db.collection.findOneAndUpdate()` 与 `upsert: true` 选项一起使用时。
- `db.collection.findOneAndReplace()` 与 `upsert: true` 选项一起使用时。
- `db.collection.bulkWrite()`

## 查询文档

### 对嵌入/嵌套文档的查询

#### 准备数据

```shell
db.inventory.insertMany( [
   { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "A" },
   { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
   { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
   { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" }
]);
```

#### 使用点符号对嵌套字段进行查询

```shell
db.inventory.find( { "size.uom": "in" } )
```

#### 使用查询运算符指定匹配

```shell
# { <field1>: { <operator1>: <value1> }, ... }
db.inventory.find( { "size.h": { $lt: 15 } } )
# AND 条件
db.inventory.find( { "size.h": { $lt: 15 }, "size.uom": "in", status: "D" } )
```

#### 匹配嵌入式/嵌套文档

不建议使用这种方式进行查询，因为需要完全匹配（包括字段顺序）。

```shell
# { <field>: <value> }
db.inventory.find( { size: { h: 14, w: 21, uom: "cm" } } )
```

### 查询数组

#### 准备数据

```shell
db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], dim_cm: [ 14, 21 ] },
   { item: "notebook", qty: 50, tags: ["red", "blank"], dim_cm: [ 14, 21 ] },
   { item: "paper", qty: 100, tags: ["red", "blank", "plain"], dim_cm: [ 14, 21 ] },
   { item: "planner", qty: 75, tags: ["blank", "red"], dim_cm: [ 22.85, 30 ] },
   { item: "postcard", qty: 45, tags: ["blue"], dim_cm: [ 10, 15.25 ] }
]);
```

#### 匹配数组

```shell
# { <field>: <value> } <value> 是要匹配的精确数组，包括元素的顺序。
db.inventory.find( { tags: ["red", "blank"] } )

# 如果想查找同时包含 "red" 和 "blank" 元素的数组，而不考虑顺序或数组中的其他元素，则请使用 $all 运算符：
db.inventory.find( { tags: { $all: ["red", "blank"] } } )
```

#### 查询数组元素

```shell
# 要查询数组字段是否至少包含一个具有指定值的元素，请使用筛选器 { <field>: <value> }，其中的 <value> 是元素值
db.inventory.find( { tags: "red" } )

# 要对数组字段中的元素指定条件 { <array field>: { <operator1>: <value1>, ... } }
db.inventory.find( { dim_cm: { $gt: 25 } } )

# 为数组元素指定多个条件
db.inventory.find( { dim_cm: { $gt: 15, $lt: 20 } } )

# 查询满足多个条件的数组元素
# 使用 $elemMatch 运算符为数组的元素指定多个条件，以使至少一个数组元素满足所有指定的条件。
db.inventory.find( { dim_cm: { $elemMatch: { $gt: 22, $lt: 30 } } } )

# 按数组索引位置查询元素
# 使用点符号，可以在数组的特定索引或位置为元素指定查询条件。
# 该数组使用从零开始的索引。
db.inventory.find( { "dim_cm.1": { $gt: 25 } } )

# 按数组长度查询数组
# 使用 $size 操作符以便按元素个数来查询数组。
db.inventory.find( { "tags": { $size: 3 } } )
```

### 查询嵌入式文档数组

#### 准备数据

```shell
db.inventory.insertMany( [
   { item: "journal", instock: [ { warehouse: "A", qty: 5 }, { warehouse: "C", qty: 15 } ] },
   { item: "notebook", instock: [ { warehouse: "C", qty: 5 } ] },
   { item: "paper", instock: [ { warehouse: "A", qty: 60 }, { warehouse: "B", qty: 15 } ] },
   { item: "planner", instock: [ { warehouse: "A", qty: 40 }, { warehouse: "B", qty: 5 } ] },
   { item: "postcard", instock: [ { warehouse: "B", qty: 15 }, { warehouse: "C", qty: 35 } ] }
]);
```

#### 查询

```shell
# 以下示例选择 instock 数组中的元素与指定文档匹配的所有文档：
db.inventory.find( { "instock": { warehouse: "A", qty: 5 } } )

# 在文档数组中嵌入的字段上指定查询条件
# 使用点符号，可以在数组的特定索引或位置为文档中的字段指定查询条件。
db.inventory.find( { 'instock.0.qty': { $lte: 20 } } )

# 为文档数组指定多个条件
# 对嵌套在文档数组中的多个字段指定条件时，可指定查询，以使单个文档满足这些条件，或使数组中任意文档（包括单个文档）的组合满足这些条件。

# 单个嵌套文档满足嵌套字段的多个查询条件
# 使用 $elemMatch 操作符在大量嵌入式文档中指定多个条件，以使至少一个嵌入式文档满足所有指定条件。
db.inventory.find( { "instock": { $elemMatch: { qty: 5, warehouse: "A" } } } )
```

### 查询返回的项目字段

#### 准备数据

```shell
db.inventory.insertMany( [
  { item: "journal", status: "A", size: { h: 14, w: 21, uom: "cm" }, instock: [ { warehouse: "A", qty: 5 } ] },
  { item: "notebook", status: "A",  size: { h: 8.5, w: 11, uom: "in" }, instock: [ { warehouse: "C", qty: 5 } ] },
  { item: "paper", status: "D", size: { h: 8.5, w: 11, uom: "in" }, instock: [ { warehouse: "A", qty: 60 } ] },
  { item: "planner", status: "D", size: { h: 22.85, w: 30, uom: "cm" }, instock: [ { warehouse: "A", qty: 40 } ] },
  { item: "postcard", status: "A", size: { h: 10, w: 15.25, uom: "cm" }, instock: [ { warehouse: "B", qty: 15 }, { warehouse: "C", qty: 35 } ] }
]);
```

#### 查询

```shell
# 仅返回指定字段和 _id 字段
# 通过在投影文档中将<field>设置为1，投影可以显式包含多个字段。
db.inventory.find( { status: "A" }, { item: 1, status: 1 } )

# 抑制 _id 字段
db.inventory.find( { status: "A" }, { item: 1, status: 1, _id: 0 } )

# 返回除已排除字段之外的所有字段
# 除 _id 字段之外，您无法在投影文档中合并包含与排除声明。
db.inventory.find( { status: "A" }, { status: 0, instock: 0 } )

# 返回嵌入式文档中的特定字段
# 您可以返回嵌入式文档中的特定字段。使用点符号引用嵌入式字段，并在投影文档中设为 1。
db.inventory.find(
   { status: "A" },
   { item: 1, status: 1, "size.uom": 1 }
)

# 抑制嵌入式文档中的特定字段
db.inventory.find(
   { status: "A" },
   { "size.uom": 0 }
)

# 对数组中嵌入式文档的投影
db.inventory.find( { status: "A" }, { item: 1, status: 1, "instock.qty": 1 } )

# 已返回数组中特定于项目的数组元素
# 对于包含数组的字段，MongoDB 提供了用来操作数组的以下投影操作符：$elemMatch、$slice 和 $first。
# 如下示例使用 $slice 投影操作符返回 instock 数组中的最后一个元素：
db.inventory.find( { status: "A" }, { item: 1, status: 1, instock: { $slice: -1 } } )

# 使用聚合表达式投影字段
# 您可以在查询投影中指定聚合表达式。通过使用聚合表达式，您可以投影新字段并修改现有字段的值。

# 例如，以下操作使用聚合表达式覆盖 status 字段的值，并投影新字段 area 和 reportNumber。
# todo 测试高版本
db.inventory.find(
   { },
   {
      _id: 0,
      item: 1,
      status: {
         $switch: {
            branches: [
               {
                  case: { $eq: [ "$status", "A" ] },
                  then: "Available"
               },
               {
                  case: { $eq: [ "$status", "D" ] },
                  then: "Discontinued"
               },
            ],
            default: "No status found"
         }
      },
      area: {
         $concat: [
            { $toString: { $multiply: [ "$size.h", "$size.w" ] } },
            " ",
            "$size.uom"
         ]
      },
      reportNumber: { $literal: 1 }
   }
)
```

### 查询 Null 字段或缺失字段

#### 准备数据

```shell
db.inventory.insertMany([
   { _id: 1, item: null },
   { _id: 2 }
])
```

#### 查询

```shell
# { item : null } 查询将匹配包含值为 null 的 item 字段或者不包含 item 字段的文档。
db.inventory.find( { item: null } )

# 要查询存在且不为 null的字段，请使用{ $ne : null }筛选器。
db.inventory.find( { item: { $ne : null } } )

# 类型检查
# { item : { $type: 10 } } 查询仅匹配包含 item 字段且其值为 null 的文档，即 item 字段的值为 BSON 类型 Null (BSON 类型 10）：
db.inventory.find( { item : { $type: 10 } } )

# 存在性检查
# { item : { $exists: false } } 查询匹配不包含 item 字段的文档：
db.inventory.find( { item : { $exists: false } } )

```

### 执行长期运行的快照查询

快照查询允许您读取最近单个时间点出现的数据。

从 MongoDB 5.0+ 开始，可以使用读关注（read concern）"snapshot"查询从节点上的数据。此功能提高了应用程序读取的多功能性和韧性。您无需创建数据的静态副本，将其移至单独的系统中，也无需手动隔离这些长时间运行的查询，以免干扰操作工作负载。相反，您可以对实时事务性数据库执行长时间运行的查询，同时读取一致状态的数据。

在从节点上使用读关注（read concern）"snapshot"不会影响应用程序的写入工作负载。只有应用程序读取受益于隔离到从节点的长时间运行查询。

当您需要执行以下操作时，请使用快照查询：

1. 执行多个相关查询，并确保每个查询从同一时间点读取数据。
2. 确保您从过去某个时间点读取的数据处于一致状态。

**比较本地读关注和快照读关注**

当 MongoDB 使用默认的"local"读关注（read concern）执行长时间运行的查询时，查询结果可能包含与查询同时发生的写入操作的数据。因此，查询可能会返回意外或不一致的结果。

为避免这种情况，请创建一个会话并指定读关注（read concern）"snapshot" 。使用读关注（read concern）"snapshot"时，MongoDB 以快照隔离方式运行查询，这意味着您的查询将读取最近单个时间点出现的数据。

```go
ctx := context.TODO()

sess, err := client.StartSession(options.Session().SetSnapshot(true))
if err != nil {
	return err
}
defer sess.EndSession(ctx)

var adoptablePetsCount int32
err = mongo.WithSession(ctx, sess, func(ctx context.Context) error {
	// Count the adoptable cats
	const adoptableCatsOutput = "adoptableCatsCount"
	cursor, err := db.Collection("cats").Aggregate(ctx, mongo.Pipeline{
		bson.D{{"$match", bson.D{{"adoptable", true}}}},
		bson.D{{"$count", adoptableCatsOutput}},
	})
	if err != nil {
		return err
	}
	if !cursor.Next(ctx) {
		return fmt.Errorf("expected aggregate to return a document, but got none")
	}

	resp := cursor.Current.Lookup(adoptableCatsOutput)
	adoptableCatsCount, ok := resp.Int32OK()
	if !ok {
		return fmt.Errorf("failed to find int32 field %q in document %v", adoptableCatsOutput, cursor.Current)
	}
	adoptablePetsCount += adoptableCatsCount

	// Count the adoptable dogs
	const adoptableDogsOutput = "adoptableDogsCount"
	cursor, err = db.Collection("dogs").Aggregate(ctx, mongo.Pipeline{
		bson.D{{"$match", bson.D{{"adoptable", true}}}},
		bson.D{{"$count", adoptableDogsOutput}},
	})
	if err != nil {
		return err
	}
	if !cursor.Next(ctx) {
		return fmt.Errorf("expected aggregate to return a document, but got none")
	}

	resp = cursor.Current.Lookup(adoptableDogsOutput)
	adoptableDogsCount, ok := resp.Int32OK()
	if !ok {
		return fmt.Errorf("failed to find int32 field %q in document %v", adoptableDogsOutput, cursor.Current)
	}
	adoptablePetsCount += adoptableDogsCount
	return nil
})
if err != nil {
	return err
}
```

## 更新文档

### 准备数据

```shell
db.inventory.insertMany( [
   { item: "canvas", qty: 100, size: { h: 28, w: 35.5, uom: "cm" }, status: "A" },
   { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "mat", qty: 85, size: { h: 27.9, w: 35.5, uom: "cm" }, status: "A" },
   { item: "mousepad", qty: 25, size: { h: 19, w: 22.85, uom: "cm" }, status: "P" },
   { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "P" },
   { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
   { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
   { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" },
   { item: "sketchbook", qty: 80, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "sketch pad", qty: 95, size: { h: 22.85, w: 30.5, uom: "cm" }, status: "A" }
] );
```

### 更新操作

```shell
# 要更新文档，MongoDB 提供了更新操作符（例如 $set）来修改字段值。
# 更新单个文档 db.inventory.updateOne
# 使用 $set 操作符将 size.uom 字段的值更新为 "cm"，并将 status 字段的值更新为 "P"
# 使用 $currentDate 操作符将 lastModified 字段的值更新为当前日期。如果 lastModified 字段不存在，则 $currentDate 将创建该字段。
db.inventory.updateOne(
   { item: "paper" },
   {
     $set: { "size.uom": "cm", status: "P" },
     $currentDate: { lastModified: true }
   }
)

# 更新多个文档 db.collection.updateMany()
db.inventory.updateMany(
   { "qty": { $lt: 50 } },
   {
     $set: { "size.uom": "in", status: "P" },
     $currentDate: { lastModified: true }
   }
)

# 替换文档   db.collection.replaceOne()
# 要替换除 _id 字段之外的文档的所有内容，请将全新文档作为第二个参数传递给 db.collection.replaceOne()
# 当替换文档时，替换文档必须仅包含字段/值对；即不包括更新操作符表达式。
db.inventory.replaceOne(
   { item: "paper" },
   { item: "paper", instock: [ { warehouse: "A", qty: 60 }, { warehouse: "B", qty: 40 } ] }
)

```

## 删除文档

- db.collection.deleteMany()
- db.collection.deleteOne()
  即使从集合中删除所有文档，删除操作也不会删除索引。

### 准备数据

```shell
db.inventory.insertMany( [
   { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
   { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "P" },
   { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
   { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
   { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" },
] );
```

### 删除

```shell
# 删除所有
db.inventory.deleteMany({})
# 删除所有符合条件的文档
db.inventory.deleteMany({ status : "A" })
# 删除一个符合条件的文档
db.inventory.deleteOne( { status: "D" } )
```

## 批量写入操作

`db.collection.bulkWrite()` 方法支持执行批量插入、更新和删除操作。
批量写操作可以是有序的，也可以是无序的。
对于有序的操作列表，MongoDB 以串行方式执行操作。如果在处理其中的一个写入操作期间出现错误，MongoDB 将返回而不处理列表中的任何其余写入操作。
对于无序列表的操作，MongoDB 可以并行执行操作，但不能保证这种行为。如果在处理其中一个写入操作期间出现错误，MongoDB 将继续处理列表中剩余的写入操作。
在分片集合上执行操作的有序列表通常比执行无序列表慢，因为对于有序列表，每个操作都必须等待前一个操作完成。
默认情况下，`bulkWrite()` 执行 `ordered` 操作。要指定 `unordered` 写入操作，请在选项文档中设置 `ordered : false`。

## 可重试写入

可重试写入允许 MongoDB 驱动程序在遇到网络错误或者在 副本集 或 分片集群 中找不到健康的 主 节点时自动重试某些写入操作。
可重试写入需要副本集或分片集群，并且不支持独立实例。
可重试写入需要支持文档级锁定的存储引擎，例如 WiredTiger 或 内存存储引擎。
客户端需要针对 MongoDB 3.6 或更高版本更新 MongoDB 驱动程序。
写关注为 0 时发出的写入操作不可重试。

事务提交和中止操作是可重试的写入操作。如果提交操作或中止操作遇到错误，则不管 retryWrites 是否设置为 false，MongoDB 驱动程序都会重试该操作一次。

## 可重试读取

可重试读取允许 MongoDB 驱动程序在遇到某些网络或服务器错误时，自动重试某些读取操作一次。
与 MongoDB Server 4.2 及更高版本兼容的官方 MongoDB 驱动程序支持可重试读取。
只有连接到 MongoDB Server 3.6 或更高版本时，驱动程序才能重试读取操作。

# 文本搜索

MongoDB 的文本搜索功能支持对字符串内容执行文本搜索的查询操作。要执行文本搜索，MongoDB 会使用文本索引和 $text 操作符。[参考](https://www.mongodb.com/zh-cn/docs/manual/text-search/)

# 地理空间查询

[参考](https://www.mongodb.com/zh-cn/docs/manual/geospatial-queries/)

# 读关注

通过有效使用写关注和读关注，您可以适当调整一致性和可用性保证的级别，例如等待更强的一致性保证，或者放松一致性要求以提供更高的可用性。
针对 MongoDB 3.2 或更高版本进行更新的 MongoDB 驱动程序支持指定读关注。
副本集和分片集群支持设置全局默认的读关注（read concern）。未指定显式读关注（read concern）的操作会继承全局默认的读关注（read concern）设置。

| 等级 | 说明 |
|:----|:----|
|"local"|查询从实例返回数据，不保证数据已写入副本集的多数成员|
|"available"|查询从实例返回数据，不保证数据已写入副本集的多数成员 读关注 "available" 不能与因果一致的会话和事务结合使用。|
|"majority"|该查询返回已被副本集多数成员确认的数据。即使失败，读取操作返回的文档也是持久性的。|
|"linearizable"|该查询返回的数据反映了在读取操作开始之前完成的所有成功的多数已确认的写入操作。该查询可能会等待并发执行的写入操作的数据复制到多数副本集成员之后，再返回结果。|
|"snapshot"|查询会返回最近某个特定时间点跨分片出现的多数提交数据。仅当事务以写关注 "snapshot" 提交时，读关注 "snapshot" 才起作用。|

# 写关注

写关注说明了 MongoDB 为针对独立运行的 mongod、副本集或分片集群的写入操作所请求的确认级别。在分片集群中，mongos 实例会将写关注传递给分片。
副本集和分片集群支持设置全局默认写关注。未指定显式写关注的操作会继承全局默认写关注设置。

```shell
{ w: <value>, j: <boolean>, wtimeout: <number> }
```

- w 选项，用于请求确认写入操作已传播到指定数量的 mongod 实例或带有指定标签的 mongod 实例。
- j 选项，用于请求确认写入操作已写入磁盘上日志。
- wtimeout 选项用于指定时间限制，防止写入操作无限期阻塞。

| w 值                          | 说明                                                                                         |
| :---------------------------- | :------------------------------------------------------------------------------------------- |
| "majority"                    | 要求确认写入操作已持久提交给计算出的多数承载数据的有投票权成员 `{ w: "majority" }` |
| `<number>`                    | 要求确认写入操作已传播到独立 mongod 或副本集中的主节点。`{ w: 1 }` |
| `<custom write concern name>` | 要求确认写入操作已传播到满足 settings.getLastErrorModes 中所定义自定义写关注的 tagged 成员。 |

# 聚合操作

```shell
db.orders.aggregate( [

   // Stage 1: Filter pizza order documents by pizza size
   {
      $match: { size: "medium" }
   },

   // Stage 2: Group remaining documents by pizza name and calculate total quantity
   {
      $group: { _id: "$name", totalQuantity: { $sum: "$quantity" } }
   }

] )
```

[参考](https://www.mongodb.com/zh-cn/docs/manual/aggregation/)

# 索引

索引支持在 MongoDB 中高效执行查询。如果没有索引，MongoDB 就必须扫描集合中的每个文档以返回查询结果。如果查询存在适当的索引，MongoDB 就可以使用该索引来限制其必须扫描的文档数。
索引可提高查询性能，但添加索引会影响写入操作的性能。对于写多读少的集合，由于每次插入操作都必须同时更新所有索引，因此会带来较高的索引成本。
MongoDB 在创建集合时会在 \_id 字段上创建一个唯一索引。\_id 索引可防止客户端插入两个具有相同 \_id 字段值的文档。您无法删除此索引。

索引的默认名称是索引键和索引中每个键的方向（1 或 -1）的连接，使用下划线作为分隔符。例如，在 { item : 1, quantity: -1 } 上创建的索引的名称为 item_1_quantity_-1。
索引一旦创建便无法重命名。相反，您必须删除索引并使用新名称重新创建索引。

## 索引创建删除

```shell
# 创建索引
db.collection.createIndex( { name: -1 } )
db.collection.getIndexes()

db.<collection>.createIndex(
   { <field>: <value> },
   { name: "<indexName>" }
)

# 删除索引  db.collection.dropIndex()  db.collection.dropIndexes()
db.<collection>.dropIndex("<indexName>")
db.<collection>.dropIndexes( [ "<index1>", "<index2>", "<index3>" ] )
db.<collection>.dropIndexes()

```

## 索引类型

- 单字段索引
- 复合索引
- 多键索引
  多键索引收集数组中存储的数据并进行排序。
  您无需显式指定多键类型。对包含数组值的字段创建索引时，MongoDB 会自动将该索引设为多键索引。
- 地理空间索引
- 文本索引
- 哈希索引
- 聚集索引 (Clustered Index)
  [参考](https://www.mongodb.com/zh-cn/docs/manual/core/indexes/index-types/)

## 索引属性

- 不分大小写的索引
- 隐藏索引 (Hidden Indexes)
- 部分索引
- 稀疏(Sparse)索引
- TTL 索引
- 唯一(Unique)索引

# 事务

1. 启动事务
2. 执行指定操作
3. 提交结果（或在出错时中止）

示例：

```go

// WithTransactionExample is an example of using the Session.WithTransaction function.
func WithTransactionExample(ctx context.Context) error {
	// For a replica set, include the replica set name and a seedlist of the members in the URI string; e.g.
	// uri := "mongodb://mongodb0.example.com:27017,mongodb1.example.com:27017/?replicaSet=myRepl"
	// For a sharded cluster, connect to the mongos instances; e.g.
	// uri := "mongodb://mongos0.example.com:27017,mongos1.example.com:27017/"
	uri := mtest.ClusterURI()

	clientOpts := options.Client().ApplyURI(uri)
	client, err := mongo.Connect(clientOpts)
	if err != nil {
		return err
	}
	defer func() { _ = client.Disconnect(ctx) }()

	// Prereq: Create collections.
	wcMajority := writeconcern.Majority()
	wcMajority.WTimeout = 1 * time.Second
	wcMajorityCollectionOpts := options.Collection().SetWriteConcern(wcMajority)
	fooColl := client.Database("mydb1").Collection("foo", wcMajorityCollectionOpts)
	barColl := client.Database("mydb1").Collection("bar", wcMajorityCollectionOpts)

	// Step 1: Define the callback that specifies the sequence of operations to perform inside the transaction.
	callback := func(sesctx context.Context) (interface{}, error) {
		// Important: You must pass sesctx as the Context parameter to the operations for them to be executed in the
		// transaction.
		if _, err := fooColl.InsertOne(sesctx, bson.D{{"abc", 1}}); err != nil {
			return nil, err
		}
		if _, err := barColl.InsertOne(sesctx, bson.D{{"xyz", 999}}); err != nil {
			return nil, err
		}

		return nil, nil
	}

	// Step 2: Start a session and run the callback using WithTransaction.
	session, err := client.StartSession()
	if err != nil {
		return err
	}
	defer session.EndSession(ctx)

	result, err := session.WithTransaction(ctx, callback)
	if err != nil {
		return err
	}
	log.Printf("result: %v\n", result)
	return nil
}

```

[参考](https://www.mongodb.com/zh-cn/docs/manual/core/transactions/)

# Go

MongoDB 为我们提供了各种语言的驱动库。使用 `go get` 进行下载即可使用。

```sh
go get go.mongodb.org/mongo-driver/mongo
```

## 连接数据库

后续所有操作都会建立在这个基础之上，需要确保这一步的正确性。

```go
package main

import (
	"context"
	"fmt"
	"time"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)
// mongo访问地址
// 我们还可以加上一些 如默认的读写关注 是否重写等 详细情况请参考官网
// 这里为了简单就用最简洁的方式进行
const uri = "mongodb://192.168.56.101:27017"

func main() {
	opts := options.Client().ApplyURI(uri)
	opts.SetAuth(options.Credential{
		// AuthSource: "admin",  // 指定认证数据库，默认为admin
		Username: "zero",  // 你创建的用户名
		Password: "123456",
	})
	opts.SetMaxPoolSize(5)
	client, err := mongo.Connect(context.TODO(), opts)
	if err != nil {
		panic(err)
	}

	defer func() {
		if err := client.Disconnect(context.TODO()); err != nil {
			panic(err)
		}
	}()

	// 检查连接情况 超时时间2s
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
	defer cancel()
	if err := client.Ping(ctx, nil); err != nil {
		panic(err)
	}
	fmt.Println("connect success")
	// 获取库存集合 后面的代码都会使用这个集合
	coll := client.Database("my-page").Collection("inventory")
	_ = coll
}
```

## 插入文档

为了方便理解把当前模型用结构体表示出来

```go
type Size struct {
	Height float64 `bson:"h,omitempty"`
	Width  float64 `bson:"w,omitempty"`
	Unit   string  `bson:"uom,omitempty"`
}

type Canvas struct {
	Quantity int      `bson:"qty,omitempty"`
	Tags     []string `bson:"tags,omitempty"`
	Size     Size     `bson:"size,omitempty"`
}

canvas := Canvas{
   Quantity: 100,
   Tags:     []string{"cotton"},
   Size: Size{
      Height: 100,
      Width:  35.5,
      Unit:   "cm",
   },
}
```

- 插入一条数据

```go
// 插入一条数据
tags := make(bson.A, 0, len(canvas.Tags))
for _, tag := range canvas.Tags {
   tags = append(tags, tag)
}
// result, err := coll.InsertOne(context.Todo(), canvas)
// 这样写也可以，但是需要调整为结构体添加一个item字段
result, err := coll.InsertOne(context.TODO(),
   bson.D{
      {"item", "canvas"},
      {"qty", canvas.Quantity},
      {"tags", tags},
      {"size", bson.D{
         {"h", canvas.Size.Height},
         {"w", canvas.Size.Width},
         {"uom", canvas.Size.Unit},
      }},
   })
if err != nil {
   fmt.Println(err)
   return
}
fmt.Println(result.InsertedID)
```

**使用结构体直接插入**

```go
type Canvas2 struct {
	Item     string   `bson:"item,omitempty"`
	Quantity int      `bson:"qty,omitempty"`
	Tags     []string `bson:"tags,omitempty"`
	Size     Size     `bson:"size,omitempty"`
}
```

```go
canvas2 := Canvas2{
   Item:     "canvas2",
   Quantity: canvas.Quantity,
   Tags:     canvas.Tags,
   Size:     canvas.Size,
}
result, err := coll.InsertOne(context.TODO(), canvas2)
```

我们往往还会创建两个方法实现 Canvas 和 Canvas2 之间的相互转换。

```go
func (c *Canvas) ToCanvas2() Canvas2 {
	return Canvas2{
		Item:     "canvas2",
		Quantity: c.Quantity,
		Tags:     c.Tags,
		Size:     c.Size,
	}
}

func (c *Canvas2) ToCanvas() Canvas {
	return Canvas{
		Quantity: c.Quantity,
		Tags:     c.Tags,
		Size:     c.Size,
	}
}
```

- 插入多条数据，类比单条插入可以很容易理解。

```go
result, err := coll.InsertMany(
	context.TODO(),
	[]interface{}{
		bson.D{
			{"item", "journal"},
			{"qty", int32(25)},
			{"tags", bson.A{"blank", "red"}},
			{"size", bson.D{
				{"h", 14},
				{"w", 21},
				{"uom", "cm"},
			}},
		},
		bson.D{
			{"item", "mat"},
			{"qty", int32(25)},
			{"tags", bson.A{"gray"}},
			{"size", bson.D{
				{"h", 27.9},
				{"w", 35.5},
				{"uom", "cm"},
			}},
		},
		bson.D{
			{"item", "mousepad"},
			{"qty", 25},
			{"tags", bson.A{"gel", "blue"}},
			{"size", bson.D{
				{"h", 19},
				{"w", 22.85},
				{"uom", "cm"},
			}},
		},
	})

```

## 查询

- 查找匹配条件的数据

```go
result := coll.FindOne(
    context.TODO(),
    bson.D{{"item", "canvas"}},
)
canvas := Canvas{}
err = result.Decode(&canvas)
```

- 批量查找

```go
cursor, err := coll.Find(context.TODO(), bson.D{{"item", "canvas"}})
if err != nil {
    fmt.Println(err)
    return
}

// 逐个解码
canvas := Canvas{}
for cursor.Next(context.TODO()) {
   err = cursor.Decode(&canvas)
   if err != nil {
      fmt.Println(err)
      return
   }
   fmt.Println(canvas)
}

// 另一种解码方式
var canvas []Canvas
err = cursor.All(context.TODO(), &canvas)
if err != nil {
   fmt.Println(err)
   return
}
```

- 指定需要返回的字段

  `SetProjection` 中 0 表示不返回，1 表示返回。其中 `_id` 是默认返回项，需要显式禁止返回。

```go
cursor, err := coll.Find(
   context.TODO(),
   bson.D{{"item", "canvas"}},
   options.Find().SetProjection(
      bson.D{
         {"_id", 0},
         {"size", 1},
         {"qty", 1},
      },
   ),
)
```

# 常用操作符汇总

- 比较操作符

  | 标记   | 作用     |
  | :----- | :------- |
  | `$eq`  | 等于     |
  | `$gt`  | 大于     |
  | `$gte` | 大于等于 |
  | `$in`  | 包含     |
  | `$lt`  | 小于     |
  | `$lte` | 小于等于 |
  | `$ne`  | 不等于   |
  | `$nin` | 不包含   |

- 逻辑操作符

  | 标记   | 作用     |
  | :----- | :------- |
  | `$and` | 并且     |
  | `$not` | 非       |
  | `$nor` | 都不满足 |
  | `$or`  | 或       |

- 元素操作符

  | 标记      | 作用                                                                                   |
  | :-------- | :------------------------------------------------------------------------------------- |
  | `$exists` | 存在                                                                                   |
  | `$type`   | [类型](https://www.mongodb.com/docs/manual/reference/bson-types/#std-label-bson-types) |

- 数组查询操作符

  | 标记         | 作用                             | 示例                                                 |
  | :----------- | :------------------------------- | :--------------------------------------------------- |
  | `$all `      | 数组中包含所有列出元素           | `{ tags: { $all: [ "ssl" , "security" ] } }`         |
  | `$elemMatch` | 数组中至少一个元素满足所有指定条件 | `{ results: { $elemMatch: { $gte: 80, $lt: 85 } } }` |
  | `$size`      | 限定数组长度                     | `{ field: { $size: 1 } }`                            |

- 数组更新操作符

  | 标记        | 作用                   | 示例                                                            |
  | :---------- | :--------------------- | :-------------------------------------------------------------- |
  | `$addToSet` | 加入集合 去重 | `{ $addToSet: { colors:"mauve" } }` |
  | `$pop`      | 移除队首或队尾         | `db.students.updateOne( { _id: 1 }, { $pop: { scores: -1 } } )` |
  | `$pull`     | 移除所有满足条件的     | `{ $pull: { fruits: { $in: [ "apples", "oranges" ] }}}`         |
  | `$push`     | 插入一条数据           | `{ $push: { scores: { $each: [ 90, 92, 85 ] } } }`              |
  | `$pullAll`  | 移除所有值匹配的       | `{ $pullAll: { scores: [ 0, 5 ] } }`                            |
  | `$sort`     | 使用数组中对象字段排序 | `$sort: { score: 1 }`                                           |

- 查询结果操作符

  | 标记     | 作用             | 使用                                     |
  | :------- | :--------------- | :--------------------------------------- |
  | `$slice` | 限定数组返回数量 | `{ <arrayField>: { $slice: <number> } }` |

- 更新操作符

  | 标记           | 作用                 |
  | :------------- | :------------------- |
  | `$currentDate` | 当前日期             |
  | `$inc`         | 递增                 |
  | `$min`         | 取较小值             |
  | `$max`         | 取较大值             |
  | `$mul`         | 乘法赋值             |
  | `$rename`      | 重命名字段           |
  | `$set`         | 设置字段             |
  | `$setOnInsert` | 插入时设置字段值     |
  | `$unset`       | 移除字段             |

- 其他操作符

  | 标记    | 作用     | 使用                                          |
  | :------ | :------- | :-------------------------------------------- |
  | `$expr` | 表达式匹配 | `{ $expr: { <expression> } } `                |
  | `$mod`  | 取余     | `{ field: { $mod: [ divisor, remainder ] } }` |

# 参考资料

[官方文档](https://www.mongodb.com/docs/manual/)
