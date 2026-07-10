---
date: "2026-01-06T20:02:13+08:00"
title: "Redis -- 入门篇"
tags: ["Database", "Redis"]
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

## 前言

Redis（Remote Dictionary Server）是一款开源的内存键值存储系统，以高性能、丰富的数据结构和完善的企业级生态著称。本文从零开始，带你完成 Redis 的安装、连接和五种基础数据类型的实战操作。

<!--more-->

## 一、安装 Redis

本文采用 Docker Compose 方式启动 Redis，无需复杂的本地编译。

### Docker Compose 配置

```yml
version: "3.8"

services:
  redis:
    image: redis:6
    container_name: redis6
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - ./data:/data
      # 可选：挂载自定义配置
      # - ./conf/redis.conf:/etc/redis/redis.conf
    command: redis-server
    environment:
      - TZ=Asia/Shanghai
    networks:
      - redis-network

networks:
  redis-network:
    driver: bridge
```

启动服务：

```bash
docker compose up -d
```

> 如需本地安装，可参考：[Redis 官方文档 -- Getting Started](https://redis.io/docs/latest/get-started/)

## 二、连接 Redis

### 方式一：进入容器（推荐）

`redis:6` 镜像内置了 `redis-cli`，可直接进入容器操作：

```bash
docker compose exec -it redis bash
redis-cli
```

### 方式二：使用 redis-cli 连接远程服务器

```bash
redis-cli [OPTIONS]
```

常用选项速查表：

| 选项 | 作用 | 示例 |
|:--|:--|:--|
| `-h/--host` | 指定连接的 IP/域名 | `redis-cli -h 192.168.1.100` |
| `-p/--port` | 指定端口 | `redis-cli -p 6380` |
| `-a/--password` | 指定密码 | `redis-cli -a 123456` |
| `--user` | ACL 用户名（Redis 6.0+，需配合 `-a`） | `redis-cli --user myuser -a mypwd` |
| `--tls` | 启用 TLS 加密 | `redis-cli --tls -h redis.example.com` |
| `-n/--database` | 指定数据库编号 | `redis-cli -n 1` |
| `-c` | 启用集群模式 | `redis-cli -c -h 192.168.1.100 -p 7000` |
| 直接传命令 | 执行单条命令后退出（无需 `-c`） | `redis-cli GET test` |

### 验证连接

```bash
127.0.0.1:6379> PING
PONG
```

返回 `PONG` 即连接成功。

## 三、键（Key）操作

Redis 是一个键值对数据库，所有数据都通过 Key 来索引。掌握 Key 操作是后续学习所有数据类型的基础。

> 在 redis-cli 中可使用 `help @generic` 快速查看通用命令文档。

| 命令 | 作用 | 示例 |
|:--|:--|:--|
| `EXISTS key` | 检查键是否存在 | `EXISTS user:1` |
| `DEL key` | 删除键值对 | `DEL user:1` |
| `TYPE key` | 获取值的数据类型 | `TYPE user:1` |
| `EXPIRE key seconds` | 设置过期时间（秒） | `EXPIRE user:1 60` |
| `EXPIREAT key timestamp` | 在指定时间戳过期 | `EXPIREAT user:1 1767931451` |
| `PERSIST key` | 移除过期时间 | `PERSIST user:1` |
| `TTL key` | 查看剩余过期秒数 | `TTL user:1` |
| `KEYS pattern` | 模糊匹配键 | `KEYS user:*` |
| `RENAMENX key newkey` | 仅当新名不存在时改名 | `RENAMENX user:1 user:2` |

## 四、五种核心数据类型

### 1. 字符串 String

String 是 Redis 最基础的数据类型，可存储文本、序列化对象、二进制数据，也常用于计数器和分布式锁。

> 默认配置下，单个 String 最大 512MB。

#### SET / GET — 基础读写

```bash
SET bike:1 Deimos
# OK

GET bike:1
# "Deimos"
```

#### NX / XX — 条件写入

```bash
SET bike:1 bike NX
# (nil)  — bike:1 已存在，写入失败

SET bike:1 bike XX
# OK     — bike:1 已存在，更新成功
```

`NX` 模式常用于实现**分布式锁**。

#### MSET / MGET — 批量操作

```bash
MSET bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth"
# OK

MGET bike:1 bike:2 bike:3
# 1) "Deimos"
# 2) "Ares"
# 3) "Vanth"
```

#### INCR / INCRBY — 原子计数器

```bash
SET total_crashes 0
INCR total_crashes
# (integer) 1

INCRBY total_crashes 10
# (integer) 11
```

支持 `DECR`、`DECRBY` 做减法操作。

---

### 2. 哈希 Hash

Hash 存储一组 field-value 对，适合存储一个对象的多个属性。可存储约 42 亿个字段，上限仅受内存限制。

#### HSET / HGET — 单字段操作

```bash
HSET bike:1 model Deimos brand Ergonom type 'Enduro bikes' price 4972
# (integer) 4

HGET bike:1 model
# "Deimos"
```

#### HMGET / HVALS — 多字段获取

```bash
HMGET bike:1 model brand
# 1) "Deimos"
# 2) "Ergonom"
```

#### HKEYS / HGETALL — 获取全部信息

```bash
HKEYS bike:1
# 1) "model"
# 2) "brand"
# 3) "type"
# 4) "price"

HGETALL bike:1
# 1) "model"    2) "Deimos"
# 3) "brand"    4) "Ergonom"
# 5) "type"     6) "Enduro bikes"
# 7) "price"    8) "4972"
```

#### HINCRBY — 字段级计数器

```bash
HINCRBY bike:1 price 100
# (integer) 5072

HINCRBY bike:1 price -100
# (integer) 4972
```

---

### 3. 列表 List

List 是一个有序的 String 链表，适合实现队列、栈，支持阻塞操作。可存储约 42 亿个元素。

#### LPUSH / LPOP — 头部入栈出栈

```bash
LPUSH bikes:repairs bike:1
# (integer) 1

LPOP bikes:repairs
# "bike:1"
```

#### LRANGE — 范围查询

```bash
RPUSH bikes:repairs bike:1 bike:2 bike:3
# (integer) 3

LRANGE bikes:repairs 0 -1
# 1) "bike:1"
# 2) "bike:2"
# 3) "bike:3"
```

索引支持负数：`-1` 表示最后一个元素。

#### LTRIM — 裁剪保留

```bash
LTRIM bikes:repairs 0 1
# 保留前两个元素，其余删除
```

#### LLEN — 获取长度

```bash
LLEN bikes:repairs
# (integer) 2
```

#### 阻塞命令

`BLPOP`、`BLMOVE` 等阻塞命令在列表为空时会等待，直到有新数据加入。适用于**生产者/消费者**模型。

---

### 4. 集合 Set

Set 是无序的唯一 String 集合，适合去重、聚类、集合运算（交、并、差）。可存储约 42 亿个元素。

#### SADD — 添加元素

```bash
SADD bikes:racing:france bike:1
# (integer) 1

SADD bikes:racing:france bike:1
# (integer) 0  — 重复添加，集合不变
```

#### SREM — 移除元素

```bash
SREM bikes:racing:france bike:1
# (integer) 1
```

#### SISMEMBER — 判断是否包含

```bash
SISMEMBER bikes:racing:france bike:1
# (integer) 1  — 存在

SISMEMBER bikes:racing:france bike:99
# (integer) 0  — 不存在
```

#### SINTER — 交集

```bash
SADD bikes:racing:france bike:1
SADD bikes:racing:usa bike:1 bike:4

SINTER bikes:racing:france bikes:racing:usa
# 1) "bike:1"
```

#### SCARD — 集合大小

```bash
SCARD bikes:racing:usa
# (integer) 2
```

---

### 5. 有序集合 Sorted Set

Sorted Set 与 Set 的区别在于每个元素关联一个 Score，按 Score 排序。Score 相同时按字典序排列。适合排行榜、滑动窗口限流等场景。

#### ZADD — 添加带分数的元素

```bash
ZADD racer_scores 10 "Norem" 12 "Castilla" 8 "Sam-Bodden"
# (integer) 3

ZADD racer_scores 10 "Royce" 6 "Ford" 14 "Prickett"
# (integer) 3
```

#### ZRANGE / ZREVRANGE — 按排名获取

```bash
ZRANGE racer_scores 0 -1
# 1) "Ford"  2) "Sam-Bodden"  3) "Norem"
# 4) "Royce" 5) "Castilla"    6) "Prickett"

ZREVRANGE racer_scores 0 2
# 1) "Prickett"  2) "Castilla"  3) "Royce"
```

#### ZRANGEBYSCORE — 按分数范围获取

```bash
ZRANGEBYSCORE racer_scores -inf 10
# 1) "Ford"  2) "Sam-Bodden"  3) "Norem"  4) "Royce"
```

#### ZRANK / ZREVRANK — 获取排名

```bash
ZRANK racer_scores "Norem"
# (integer) 2  — 升序排名第 3

ZREVRANK racer_scores "Norem"
# (integer) 3  — 降序排名第 4
```

#### ZREM / ZREMRANGEBYSCORE — 删除元素

```bash
ZREM racer_scores "Castilla"
# (integer) 1

ZREMRANGEBYSCORE racer_scores -inf 9
# (integer) 2  — 删除分数 ≤ 9 的元素
```

## 五、总结

本文覆盖了 Redis 入门最核心的内容：

1. **环境搭建**：Docker Compose 一键启动
2. **连接管理**：redis-cli 参数速查
3. **Key 操作**：通用的键管理命令
4. **五种数据类型**：String、Hash、List、Set、Sorted Set 的完整操作

掌握这些内容后，你已经可以使用 Redis 完成绝大多数日常开发任务。想深入了解 Redis 的持久化机制、高可用架构和缓存策略，请继续阅读 [Redis 进阶篇：从架构到实战](./redis_advanced.md)。

## 参考资料

- [Redis 官方文档](https://redis.io/docs/latest/)
- [Redis GitHub 仓库](https://github.com/redis/redis)
- [菜鸟教程 Redis](https://www.runoob.com/redis/redis-tutorial.html)
