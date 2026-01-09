---
date: "2026-01-06T20:02:13+08:00"
title: "Redis -- 基础使用"
tags: ["Database", "Redis"]
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

`Redis（Remote Dictionary Server`）是一款开源的内存键值存储系统，常用作缓存、数据库和消息中间件，以高性能、丰富数据结构和完善的高可用生态著称，广泛应用于互联网后端与分布式场景。本文主要介绍 `Redis` 的基本使用方法。

<!--more-->

## 安装

本文采用 `docker compose` 启动 `redis` 服务器。

> 如需本地安装可以参考：  
> [菜鸟教程 -- Redis 安装](https://www.runoob.com/redis/redis-install.html)  
> [Redis 官方文档 -- getstart](https://redis.io/docs/latest/get-started/)

使用的 `docker compose` 文件如下：

```yml
version: "3.8" # Compose版本，建议使用3.8+

services:
  redis:
    # Redis镜像（指定版本，避免使用latest导致版本不一致）
    image: redis:6
    # 容器名称（自定义）
    container_name: redis6
    # 重启策略（容器挂掉自动重启）
    restart: always
    # 端口映射：宿主机端口:容器内端口
    ports:
      - "6379:6379"
    # 挂载卷（核心：本地目录映射到容器内）
    volumes:
      # 本地数据目录映射到容器内的/data
      - ./data:/data

      # 使用本地配置  本地配置文件映射到容器内的/etc/redis/redis.conf
      # - ./conf/redis.conf:/etc/redis/redis.conf
    # 启动命令（指定使用自定义的配置文件）

    # 使用本地配置文件
    # command: redis-server /etc/redis/redis.conf

    # 使用内置配置
    command: redis-server
    # 环境变量（可选，设置时区，避免日志时间错乱）
    environment:
      - TZ=Asia/Shanghai
    # 网络模式（默认bridge即可）
    networks:
      - redis-network

# 自定义网络（可选，隔离容器网络）
networks:
  redis-network:
    driver: bridge
```

## 连接 redis

1. 使用本文方式启动 `redis server` 的情况
   由于 `redis:6` 镜像中内置了，`redis-cli` 所以我们可以直接进入容器使用

   ```bash
   # 1. 进入 redis 容器
   docker compose exec -it redis bash

   # 2. 使用容器内置的 redis-cli 连接
   redis-cli
   ```

2. 使用 `redis-cli` 连接已有 `redis-server`
   语法模板如下：

   ```bash
   redis-cli [OPTIONS]
   ```

   如过不使用 `[OPTIONS]` 默认连接本地 6379 端口。

   | 选项          | 作用                                           | 示例                                                               |
   | :------------ | :--------------------------------------------- | :----------------------------------------------------------------- |
   | -h/--host     | 指定连接的 ip/域名                             | redis-cli -h 127.0.0.1（本地）、redis-cli -h 192.168.1.100（远程） |
   | -p/--port     | 指定连接的端口                                 | redis-cli -p 6380（非默认 6379 端口）                              |
   | -a/--password | 指定连接密码（对应 Redis 的 requirepass）      | redis-cli -a 123456                                                |
   | --user        | Redis 6.0+ ACL 用户名（需配合密码）            | redis-cli -u myuser -a mypwd                                       |
   | --tls         | 启用 TLS/SSL 加密连接,一些云厂商会默认开启加密 | redis-cli --tls -h redis.example.com -p 6379                       |
   | -n/--database | 指定要连接的数据库                             | redis-cli -n 1（直接进入 1 号库）                                  |
   | -c/--command  | 直接执行命令并退出（无需进入交互模式）         | redis-cli GET test（查询 test 键的值）                             |

进入 `redis-cli` 的交互界面后，我们可以通过 `PING` 命令测试连接

```bash
root@b970370ace80:/data# redis-cli
127.0.0.1:6379> PING
PONG
```

返回 PONG 说明连接成功了

## 数据类型及操作

`Redis` 提供了多种数据类型，这里对最常使用的基础练习 `字符串(String)`、`哈希(Hash)`、`列表(list)`、`集合(Set)`、`有序集合(Sorted Set)` 进行介绍。

### 键 Key

`Redis` 是一个 `键值对（Key-Value）` 数据库（字典），其中的 `Key` 就是键。键的数据类型是字符串，通过对 `Key` 进行哈希散列可以找到 `Value` 的位置。所有 `Key` 的操作都是对 `键值对` 整体的操作。

常用命令：

> 在 `redis-cli` 交互界面可以使用 `help @generic` 查看官方提供的键命令文档

| 命令                     | 作用                                   | 示例                       |
| :----------------------- | :------------------------------------- | :------------------------- |
| `DEL key [key]...`       | 删除一个或多个键值对                   | `DEL key1`                 |
| `EXISTS key [key]...`    | 查看一个或多个键值对是否存在           | `EXISTS key1`              |
| `EXPIRE key seconds`     | 设置键值对 n 秒后过期                  | `EXPIRE key1 5`            |
| `EXPIREAT key timestamp` | 设置键值对在指定 unix 时间戳过期       | `EXPIREAT key1 1767931451` |
| `KEYS pattern`           | 模糊匹配键 \*匹配任意字符串            | `KEYS *` , `KEYS ke*`      |
| `PERSIST key`            | 移除键值对的过期时间，键值对将持久保持 | `PERSIST key1`             |
| `TTL key`                | 查看键值对几秒后过期                   | `TTL key1`                 |
| `RENAMENX key newkey`    | 当行键名不存在时，修改键名             | `RENAMENX key1 ke2`        |
| `TYPE key`               | 获取键值对 值的数据类型                | `TYPE key1`                |

### 字符串 String

`String` 数据类型存储一串字节序列，包括文本，序列化的对象，二进制数组等。`String` 类型通常被用作数据缓存。事实上，它他还可以用作计数器，也可以进行位运算。

> [!important] 默认配置下，`String` 类型数据单挑长度不能超过 512MB

一下是一些常用命令:

- `SET`  
  用来存储 `String` 值，效果类似赋值，可以通过添加 `NX` 参数实现仅当键不存在时，执行操作。

  ```bash
  > set bike:1 Deimos
  OK
  > set bike:1 bike nx
  (nil)
  > set bike:1 bike xx
  OK
  ```

  添加 `NX` 参数时，常作做枷锁操作。

- `SETNX`

  同 `SET` 添加 `NX` 参数

- `GET`

  获取存储的 `String` 值。

  ```bash
  > GET bike:1
  "Deimos"
  ```

- `MSET`

  一次设置多个键存储的 `String` 值。

  ```bash
  > mset bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth"
  OK
  ```

- `MGET`

  一次获取多个键存储的 `String` 值。

  ```bash
  > mget bike:1 bike:2 bike:3
  1) "Deimos"
  2) "Ares"
  3) "Vanth"
  ```

- `INCR`

  计数器操作，为当前存贮值加 1（默认值为 0）

  ```bash
  > set total_crashes 0
  OK
  > incr total_crashes
  (integer) 1
  ```

- `INCRBY`

  为当前存储值加上指定值，可以通过加负数实现减（`DECR` 和 `DECRBY` 命令）效果。

  ```bash
  > incrby total_crashes 10
  (integer) 11
  ```

### 哈希 Hash

`Hash` 存储的是一组键值对(这里叫作 `field-value pairs`)的数据类型，可以理解为 `String` 的集合。`Hash` 常用类存储一个对象（每个字段对应一个属性）。

> `Hash` 可以存储 `4,294,967,295 (2^32 - 1)` 个键值对，几乎可以认为它的存储上限仅取决于可使用的内存。

常用命令：

- `HSET`

  存储一个或多个字段值对(`field-value pairs`)。

  ```bash
  # hset key field value [field value ...]
  > HSET bike:1 model Deimos brand Ergonom type 'Enduro bikes' price 4972
  (integer) 4
  ```

- `HGET`、`HMGET`、`HVALS`

  分别是获取一个字段值，多个字段值，所有字段值

  ```bash
  > HGET bike:1 model
  "Deimos"
  > HGET bike:1 price
  "4972"
  > HMGET bike:1 model brand
  1) "Deimos"
  2) "Ergonom"
  ```

- `HKEYS`

  获取所有的字段名。

  ```bash
  > HKEYS bike:1
  1) "model"
  2) "brand"
  3) "type"
  4) "price"
  ```

- `HGETALL`

  获取所有的 字段值对。

  ```bash
  > HGETALL bike:1
  1) "model"
  2) "Deimos"
  3) "brand"
  4) "Ergonom"
  5) "type"
  6) "Enduro bikes"
  7) "price"
  8) "4972"
  ```

- `HINCRBY`

  计数器操作，可类比 `INCRBY` 命令理解。

  ```bash
  > HINCRBY bike:1 price 100
  (integer) 5072
  > HINCRBY bike:1 price -100
  (integer) 4972
  ```

### 列表 List

`List` 是一个 `String` 值的链表，通常被用于实现 队列、栈。`List` 支持阻塞操作。

> `List` 可以存储 `4,294,967,295 (2^32 - 1)` 个 `String` 值，同样可以认为它的存储上限仅取决于可使用的内存。

常用命令：

- `LPUSH`

  在链表头部添加数据。

  ```bash
  > LPUSH bikes:repairs bike:1
  (integer) 1
  > LPUSH bikes:repairs bike:2
  (integer) 2
  ```

- `LPOP`

  从链表头部取出数据。

  ```bash
  > LPOP bikes:repairs
  "bike:2"
  > LPOP bikes:repairs
  "bike:1"
  ```

- `LLEN `

  获取链表长度。

  ```bash
  > LLEN bikes:repairs
  (integer) 0
  ```

- `LRANGE`

  从链表中查询出一连串数据，后接入两个值 `start`、`stop`，这两个值会被解析为链表索引，索引值为链表长度加上对应值，如：

  如果链表长度、`start`、`stop` 分别为 2, 0, -1,则 `start` 代表索引为 0 的元素（第一个元素），`stop` 为索引为 1 的元素（最后一个元素）。

  ```bash
  > LRANGE bikes:repairs 0 -1
  1) "bike:1"
  > LRANGE bikes:finished 0 -1
  1) "bike:2"
  ```

- `LTRIM`

  从链表移除一连串数据，参数解析方式同 `LRANGE` 。
  ```bash
  > RPUSH bikes:repairs bike:1 bike:2 bike:3 bike:4 bike:5
  (integer) 5
  > LTRIM bikes:repairs 0 2
  OK
  ```


### 集合 Set



### 有序集合 Sorted Set

## 参考资料

[Redis 官方文档](https://redis.io/docs/latest/)  
[Redis Github 仓库](https://github.com/redis/redis)  
[菜鸟教程 Redis](https://www.runoob.com/redis/redis-tutorial.html)
