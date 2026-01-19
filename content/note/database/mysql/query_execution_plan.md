---
date: "2026-01-19T15:57:46+08:00"
title: "MySQL -- 执行计划"
tags: ["MySQL", "execution plan"]
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

`MySQL` 在真正执行你的 `SQL` 之前，会先分析这条 `SQL` 的写法、表的索引、数据量、字段类型等信息，然后制定出 `它认为最优` 的执行方案，这个方案就叫 `执行计划` 。

<!--more-->

了解执行计划，是一个优化 `MySQL` 性能的不错切入点。

## 查看执行计划

对 `SELECT`, `DELETE`, `INSERT`, `REPLACE`, `UPDATE` 语句执行 `EXPLAIN` 命令，`MySQL` 会返回优化器关于这些语句的执行计划，如：

> `EXPLAIN`, `DESC`, `DESCRIBE` 三个命令是等效的，通常情况下我们使用 `EXPLAIN` 查看执行计划。  
> `DESCRIBE` 命令用于查看表结构。
> `DESC` 语言其和排序语法中的逆序语法相同，所以不常单独使用。

```mysql
-- 查看单表查询的执行计划
EXPLAIN SELECT id, name FROM user WHERE age > 20;
...
```

## 执行计划格式

`EXPLAIN` 为语句中使用到每一个使用到的表输出一行数据，输出顺序为使用顺序。输出表格包含一下字段：

| 列名          | 说明                   |
| :------------ | :--------------------- |
| id            | 标记执行优先级         |
| select_type   | SELECT类型             |
| table         | 表名                   |
| partitions    | 匹配的分片             |
| type          | 数据查询方式/类型      |
| possible_keys | 可能用到的索引         |
| key           | 实际使用的索引         |
| key_len       | 实际使用的索引长度     |
| ref           | 与索引相比较的列       |
| rows          | 预估需要查询的行数     |
| filtered      | 通过条件筛选出的行比例 |
| Extra         | 补充信息               |

### type

`type` 是性能核心指标，重中之重！`type` 的取值决定了查询的性能等级！

```plaintext
# type 的取值从优到劣
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL
```

常见 `type` 说明：

| type         | 说明                           |
| :----------- | :----------------------------- |
| const/system | 查询复杂的为常量               |
| eq_ref       | 使用主键/唯一索引匹配一行数据  |
| ref          | 非唯一索引，返回某个值的所有行 |
| range        | 范围匹配，部分使用到了索引     |
| index        | 扫描整个索引表                 |
| ALL          | 全表扫描                       |

### possibale_keys

可能被用到的 索引 字段，如果为 `NULL` 就需要考虑优化索引了，因为查询大概率会走全表扫描（ALL）。

### key

实际使用的 索引 字段，这是判断索引是否生效的依据。

### rows

优化器预估可能要扫描的行数，这个值越小越好。

### filtered

筛选比例，最大值为 100，没有被过滤掉的行。

### Extra

性能优化提示，一些需要注意的提示信息：

1. `Using filesort`：无法利用索引进行排序，将数据加载到内存/磁盘中排序，需要考虑添加索引。
2. `Using temporary`：需要创建临时表来进行去重、分组，会占用额外的性能。需要进行优化。
3. `Using join buffer`：使用了连接缓冲区，需要考虑优化。
4. `Using where`：通过索引查询数据后，任然需要根据 `where` 条件过滤。
5. `Using index condition`：使用索引下推减少了回表次数。
6. `Using index`：索引覆盖，性能极佳。

## 参考资料

[MySQL 官方文档](https://dev.mysql.com/)
