---
date: "2026-02-09T16:37:19+08:00"
title: "Proto3 基础语法"
tags: ["protobuf"]
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

`Proto3` 是 `Google Protocol Buffers`（Protobuf）的第三个版本，2016 年推出，是一种语言中立、平台中立、可扩展的结构化数据序列化机制，用于序列化结构化数据，常用于通信协议、数据存储等场景，相比 `Proto2` 更简洁、高效、跨语言兼容性更强。

<!--more-->

## 应用场景

由于 `protobug3` 其良好的跨语言兼容性、优秀的性能表现被广泛应用在以下领域：

- `RPC` 通信：如 `gRPC` 默认使用 `Proto3` 定义服务接口和消息结构，高效传输数据。
- 数据存储：序列化数据存储到磁盘或数据库，如日志存储、配置文件等。
- 跨语言数据交换：在不同语言编写的系统间传输数据，如 `Java` 后端与 `Go` 客户端通信。
- 移动开发：生成的代码体积小、效率高，适配移动端性能要求。

## 消息定义

官网中给我们提供了一个简单的例子，用来了解 `proto3` 中如何定义一个消息。假设你想要定义一个搜索请求消息格式，其中每个搜索请求都包含一个查询词字符串、你感兴趣的查询结果所在的特定页码数和每一页应展示的结果数。以下是定义请求的 `proto` 文件：

```proto
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 results_per_page = 3;
}
```

### 语法版本

> `proto3` 中注释语法同 `C` 语言中一样：使用 `//` 进行单行注释，使用 `/*  */` 进行多行注释。

第一行使用 `syntax = "proto3";` 指明使用的 `proto3` 语法。在 `proto3` 中这个版本说明需要是文件中第一个非空、非注释行。如果不指定版本，默认为 `proto2`。

### 消息定义

使用 `message` 关键字定义消息，结构如下：

```proto
message 消息名 {
  字段1;
  字段2;
  ...
}
```

### 字段定义

在 `proto3` 中定义字段需要 字段基数、字段类型、字段名称、以及字段编号等信息，形如 `[基数] 类型 名称 = 编号;`。

- 字段基数有以下类型：
  - Singular：在 `proto3` 中有两种类型下 `singular` 字段
    - optional： 这是一个比较推荐的类型。它有两种状态，已被赋值和未被赋值。仅在已赋值状态下字段会被序列化。
    - implicit： 类型的默认值，这种类型的序列化处理需要分两种情况。如果字段是一个消息，会被按照 `optional` 处理。如果不是，仅在被赋值为非零值时进行序列化。
  - repeated： 代表字段可包含零个或多个同类型元素，且顺序会保留，通常表达为数组。
  - map：键值对类型。

- 字段类型
  在上面的示例中，所有字段都是标量类型（scalar types）: 两个整数(page_number 和 result_per_page)和一个字符串(query)。但是也可以为字段指定组合类型，包括枚举和其他消息类型。

- 字段编号
  如你所见，消息定义中的每个字段都有一个唯一的编号。这些字段编号用来在消息二进制格式中标识字段，在消息类型使用后就不能再更改。字段编号需要满足以下要求：
  1. 编号值在 `1 ~ 536,870,911(2^29-1)` 之间
  2. 字段编号需唯一。
  3. 不能使用 `reserved` 关键字定义的保留字段标号。 `reserved` 会在后文中会进行介绍。
  4. 不能使用分配个 `extensions` 的字段编号。

## 基础类型

`proto3` 定义了大量常用的标量数据数据类型,下面是各种类型的说明以及在不同语言中对应的数据类型：

| Proto    | 说明                                                            | C++         | Java/Kotlin | Python                           | Go      | Ruby                           | C#         | PHP            | Dart   | Rust        |
| :------- | :-------------------------------------------------------------- | :---------- | :---------- | :------------------------------- | :------ | :----------------------------- | :--------- | :------------- | :----- | :---------- |
| double   | IEEE 754 中双精度浮点类型                                       | double      | double      | float                            | float64 | Float                          | double     | float          | double | f64         |
| float    | IEEE 754 中单精度浮点类型                                       | float       | float       | float                            | float32 | Float                          | float      | float          | double | f32         |
| int32    | 使用可变长度编码。编码负数效率低下                              | int32_t     | int         | int                              | int32   | Fixnum or Bignum (as required) | int        | integer        | int    | i32         |
| int64    | 使用可变长度编码。编码负数效率低下                              | int64_t     | long        | int/long                         | int64   | Bignum                         | long       | integer/string | Int64  | i64         |
| uint32   | 使用可变长度编码。                                              | uint32_t    | int         | int/long                         | uint32  | Fixnum or Bignum (as required) | uint       | integer        | int    | u32         |
| uint64   | 使用可变长度编码。                                              | uint64_t    | long        | int/long                         | uint64  | Bignum                         | ulong      | integer/string | Int64  | u64         |
| sint32   | 使用可变长度编码。更有效地编码负数。                            | int32_t     | int         | int                              | int32   | Fixnum or Bignum (as required) | int        | integer        | int    | i32         |
| sint64   | 使用可变长度编码。更有效地编码负数。                            | int64_t     | long        | int/long                         | int64   | Bignum                         | long       | integer/string | Int64  | i64         |
| fixed32  | 总是四个字节。如果值经常大于228，则比 uint32更有效率。          | uint32_t    | int         | int/long                         | uint32  | Fixnum or Bignum (as required) | uint       | integer        | int    | u32         |
| fixed64  | 总是8字节。如果值经常大于256，则比 uint64更有效率。             | uint64_t    | long        | int/long                         | uint64  | Bignum                         | ulong      | integer/string | Int64  | u64         |
| sfixed32 | 总是四个字节。                                                  | int32_t     | int         | int                              | int32   | Fixnum or Bignum (as required) | int        | integer        | int    | i32         |
| sfixed64 | 总是八个字节。                                                  | int64_t     | long        | int/long                         | int64   | Bignum                         | long       | integer/string | Int64  | i64         |
| bool     |                                                                 | bool        | boolean     | bool                             | bool    | TrueClass/FalseClass           | bool       | boolean        | bool   | bool        |
| string   | 字符串必须始终包含 UTF-8编码的或7位 ASCII 文本，且不能长于232。 | std::string | String      | str/unicode                      | string  | String (UTF-8)                 | string     | string         | String | ProtoString |
| bytes    | 可以包含任何不超过232字节的任意字节序列。                       | std::string | ByteString  | str (Python 2), bytes (Python 3) | []byte  | String (ASCII-8BIT)            | ByteString | string         | List   | ProtoBytes  |

## 枚举类型

在 `proto3` 中使用 `enum` 关键字定义枚举类型：

```proto
enum Corpus {
  CORPUS_UNSPECIFIED = 0;
  CORPUS_UNIVERSAL = 1;
  CORPUS_WEB = 2;
  CORPUS_IMAGES = 3;
  CORPUS_LOCAL = 4;
  CORPUS_NEWS = 5;
  CORPUS_PRODUCTS = 6;
  CORPUS_VIDEO = 7;
}

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 results_per_page = 3;
  Corpus corpus = 4;
}

```

枚举类型中第一个值总是 0，这个值会作为这个枚举类型的默认值。
此外我们可以使用 `option allow_alias = true;` 开启枚举类型的别名功能。

```proto
enum EnumAllowingAlias {
  option allow_alias = true;
  EAA_UNSPECIFIED = 0;
  EAA_STARTED = 1;
  EAA_RUNNING = 1;
  EAA_FINISHED = 2;
}

enum EnumNotAllowingAlias {
  ENAA_UNSPECIFIED = 0;
  ENAA_STARTED = 1;
  // ENAA_RUNNING = 1;  // Uncommenting this line will cause a warning message.
  ENAA_FINISHED = 2;
}

```

## 字段默认值

在使用 `proto3` 进行反序列化时，如果消息中的不包含指定字段的时候，会使用默认值进行字段填充，这个默认值会根据字段类型有所不同：

| 类型     | 默认值                                                                                            |
| :------- | :------------------------------------------------------------------------------------------------ |
| string   | 空字符串                                                                                          |
| byte     | 空字节                                                                                            |
| bool     | false                                                                                             |
| 数值类型 | 0                                                                                                 |
| 枚举类型 | 第一个定义的枚举值，具体值为0                                                                     |
| 消息类型 | 未赋值状态，确切值和具体变成语言有关，可以参考[代码生成参考手册](https://protobuf.dev/reference/) |

## 保留值

当我们移除消息中的某些字段后，如果没有做好记录，在后续的更新中这些字段的编号或名称被再次使用。此时如果老版本和新版本的 `proto` 被同时使用，可能造成包括解析异常、数据损毁等一系列严重后果。`proto3` 中使用 `reserved` 标记这些保留字段编号、字段名称，使用编译器进行误用检测。

```proto
message Product {
  int32 id = 1;
  string title = 2;
  // 预留 10 到 20 的编号范围，用于未来扩展核心字段
  reserved 10 to 20;
  float price = 3;
  // 错误示例：尝试使用预留范围的编号，编译报错
  // string desc = 10; // 编译错误：Field number 10 is reserved in "Product".
}

message User {
  int32 id = 1;
  string name = 2;
  // 原字段 3（age）、5（address）已删除，用 reserved 标记禁止复用
  reserved 3, 5;
  string email = 4;
  // 错误示例：若尝试重新定义字段 3，编译报错
  // int32 age = 3; // 编译错误：Field number 3 is reserved in "User".
}

```

`reserved` 既可以在消息中使用、也可在枚举类型中使用，但是需要注意： 不能混合编号和名称的保留。

```proto
message Example {
  // ✅ 正确：分开声明编号和名称
  reserved 2, 4 to 8;
  reserved "old_field", "deprecated_name";

  // ❌ 错误：混合编号和名称
  // reserved 2, "old_field"; // 编译错误：Cannot mix field names and numbers in a reserved statement.
}
```

## 引用

`proto3` 中可以使用 `import` 关键字引用其他 `proto` 文件，从而实现文件的复用。

```proto
import "myproject/other_protos.proto";
```

使用 `import` 时需要配合编译器参数 `-I/--proto_path` 指定引用文件目录（可以指定多个）。以下是官方文档中给出的一个示例：

```bash
my_project/
├── protos/
│   ├── main.proto
│   └── common/
│       └── timestamp.proto
```

在 `main.proto` 中引用 `timestamp.proto`：

```proto
// Located in my_project/protos/main.proto
import "common/timestamp.proto";

```

在 `my_project` 目录下运行编译器，`proto` 文件路径参数为 `--proto_path=protos`。

### package

在 `proto3` 中可以通过 `package` 关键字，指定 `proto` 文件的包名，从而避免引入文件中存在的同名消息体、同名枚举类型等问题：

```proto
package foo.bar;
message Open { ... }
```

```proto
message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}

```

## 服务定义

如果希望将消息类型与 `RPC (远程过程调用)` 系统一起使用，可以在 `proto` 文件中使用 `service` 定义服务，编译器将用你选择的语言生成服务接口代码和存根。

```proto
...
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
...

```

## 使用技巧

- 经常使用的消息字段编号尽量使用 `1～15`， 大的编号可能使用更多的编码空间，范围1到15中的字段编号需要一个字节进行编码，包括字段编号和字段类型。范围16到2047的字段编号采用两个字节。
- 根据实际情况选择合适的字段类型，详情可以参考 [基础类型](#基础类型)。
- 使用 `reserved` 保留/弃用字段编号、字段名称。
- 注意字段类型的默认值，避免默认值引起的异常。

## TODO

- Oneof
- Any
- Map
- Options

## 参考资料

[protobuf 参考文档](https://protobuf.dev/programming-guides/proto3/)
