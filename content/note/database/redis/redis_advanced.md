---
date: "2026-01-06T20:02:13+08:00"
title: "Redis -- 进阶篇"
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

如果你已经掌握了 Redis 的基本数据类型和命令操作，本篇将带你深入 Redis 的内核——从内存模型、持久化机制，到高可用架构和经典缓存问题的解决方案。这些是面试高频考点，也是生产环境必知内容。

> 建议先阅读 [Redis 入门指南：从安装到数据操作](./redis_beginner.md) 了解基础。

<!--more-->

## 一、Redis 的定位与性能

Redis 是一个开源（BSD 许可）的内存数据结构存储系统，可用作数据库、缓存和消息中间件。

**核心特点：**

- 6.0 版本后开放多线程（此前仅内部操作如磁盘持久化使用多线程）
- 支持大部分主流编程语言
- 数据存储在内存中，读写极快

**性能基准（官方 benchmark）：**

- 测试条件：50 个并发、10 万次请求、256 字节字符串
- 读：约 11 万次/秒
- 写：约 8.1 万次/秒

## 二、关系型 vs 非关系型数据库

理解 Redis 的本质，需要先理解它在数据库谱系中的位置。

### 关系型数据库（RDBMS）

采用二维表格模型，数据以行和列组织，遵循三范式设计。

| 优点 | 缺点 |
|:--|:--|
| 结构清晰，易于理解 | 磁盘 I/O 是并发瓶颈 |
| SQL 语法成熟，使用方便 | 海量数据查询效率低 |
| ACID 事务保障，数据一致性强 | 横向扩展困难，通常需要停机 |

代表：MySQL、Oracle、PostgreSQL

### 非关系型数据库（NoSQL）

结构灵活，不遵循固定 Schema，以 Key-Value 为核心。

| 优点 | 缺点 |
|:--|:--|
| 字段按需添加，无需多表联查 | 只能存储相对简单的数据 |
| 数据无耦合，易于水平扩展 | 不适合复杂查询 |
| 读写速度快（内存存储） | 不适合持久存储海量数据 |

代表：Redis（K-V）、MongoDB（文档）、Elasticsearch（搜索）、HBase（分布式）

### 对比总结

| 维度 | 关系型数据库 | 非关系型数据库 |
|:--|:--|:--|
| 存储介质 | 硬盘 | 内存（缓存） |
| 存储格式 | 基础类型 | K-V、文档、图片等 |
| 扩展性 | 有多表查询，扩展困难 | 数据无耦合，易于扩展 |
| 持久性 | 适合持久存储 | 不适合持久存储 |
| 一致性 | 强一致性（ACID） | 最终一致性 |

## 三、Redis 高级使用技巧

### 层级目录 Key

通过格式化的 Key 模拟目录结构：

```bash
SET user:1:cart1:item1 phone
```

命名规范建议：`业务:对象ID:属性:子属性`

### 失效时间管理

**存入时设置：**

```bash
SET key value EX 60    # 60 秒后过期（PX 为毫秒）
SET key value NX EX 60 # 仅当 key 不存在时设置，常用于分布式锁
```

**为已有 key 设置：**

```bash
EXPIRE key 300    # 300 秒后过期（PEXPIRE 为毫秒）
```

**查询过期状态：**

```bash
TTL key
# -1: 永不过期
# -2: key 已失效
# 正整数: 剩余秒数
```

### 存取对象

通过 JSON 或 XML 序列化对象后，以 String 类型存储：

```bash
SET user:1 '{"name":"Alice","age":25}' EX 3600
GET user:1
# '{"name":"Alice","age":25}'
```

## 四、持久化机制

Redis 是内存数据库，宕机会导致数据丢失。持久化就是将内存数据写入磁盘的过程。

### RDB（Redis Database）

按时间快照方式，将某一时刻的全部数据保存为 `dump.rdb` 文件。

**触发条件（redis.conf）：**

```conf
save 900 1      # 900 秒内至少 1 个 key 变化，自动保存
save 300 10     # 300 秒内至少 10 个 key 变化
save 60 10000   # 60 秒内至少 10000 个 key 变化
```

也可以手动触发：`SAVE`（阻塞）或 `BGSAVE`（后台非阻塞）。

| 优点 | 缺点 |
|:--|:--|
| 恢复速度快 | 可能丢失最后一次快照后的数据 |
| 文件紧凑，适合备份 | 数据量大时 fork 子进程耗时较长 |

### AOF（Append Only File）

记录每一条写命令，通过回放命令恢复数据。

**配置（redis.conf）：**

```conf
appendonly yes
appendfilename "appendonly.aof"
```

| 优点 | 缺点 |
|:--|:--|
| 实时性强，几乎不丢数据 | 文件持续增长，需要定期重写 |
| 可读性好，便于分析 | 恢复速度比 RDB 慢 |

### 混合持久化（推荐）

Redis 4.0+ 支持 RDB + AOF 混合模式：重启时先加载 RDB 快照，再回放增量 AOF 命令，兼顾恢复速度和数据安全。

## 五、高可用架构

### 主从复制

- **主节点（Master）**：负责写操作
- **从节点（Slave）**：负责读操作，数据从主节点同步

查看复制状态：`INFO replication`

| 优点 | 缺点 |
|:--|:--|
| 解决了单点故障 | 主节点宕机后，从节点可能产生脏数据 |
| 读写分离提升性能 | 多份副本占用资源 |

### Sentinel（哨兵）

Sentinel 监控主从节点，当主节点宕机时自动选举新的主节点。

```conf
sentinel monitor mymaster 192.168.10.100 6379 2
sentinel auth-pass mymaster root
sentinel down-after-milliseconds 30000     # 30 秒延迟，避免网络波动误判
sentinel failover-timeout mymaster 180000  # 3 分钟内选举失败则放弃
```

Sentinel 是 Redis 高可用的标准方案，适合中小规模集群。

### Redis Cluster

大规模部署时使用原生集群模式，数据分片存储在不同节点上。

- **原生集群**：通过 `redis-cli --cluster` 创建，手动分配槽位
- **第三方方案**：Codis 等中间层方案，客户端透明

| 优点 | 缺点 |
|:--|:--|
| 水平扩展，支持海量数据 | 配置复杂，运维成本高 |
| 自动故障转移 | 跨节点事务和 Lua 脚本受限 |

## 六、缓存三大问题

### 缓存查询流程

```
客户端请求 → 查缓存 → 命中则返回
                   → 未命中 → 查数据库 → 写入缓存 → 返回
```

### 缓存击穿

**场景**：某个热门 Key 过期，大量并发请求同时打到数据库。

**解决方案：**

1. **异步更新**：不设过期时间，使用过期标记 + 后台线程主动刷新缓存
2. **互斥锁**：缓存失效时，只允许一个线程查库并回填缓存，其余线程等待

### 缓存穿透

**场景**：查询一个缓存和数据库都不存在的数据，每次请求都打到数据库。

**解决方案：**

1. **布隆过滤器**：在缓存前拦截非法 Key，快速判断请求是否有效
2. **缓存空值**：数据库查不到也缓存一个空值，设置较短的过期时间
3. **异步更新**：无论 Key 是否存在都直接返回，后台异步读库并更新缓存（需做缓存预热）

### 缓存雪崩

**场景**：大量热门 Key 同时过期，或 Redis 整体宕机，请求全部涌入数据库。

**解决方案：**

1. **随机过期时间**：给过期时间加上随机偏移量，避免同时失效
2. **多级缓存**：设置两层缓存（A 有过期时间，B 永不过期），A 失效时从 B 读取并异步重建 A
3. **集群部署**：在不同节点分布热点 Key

## 七、内存淘汰策略

当内存不足以容纳新数据时，Redis 根据配置的策略决定淘汰哪些 Key：

| 策略 | 行为 |
|:--|:--|
| `volatile-lru`（推荐） | 从设置了过期时间的 Key 中，淘汰最近最少使用的 |
| `volatile-ttl` | 从设置了过期时间的 Key 中，淘汰最快过期的 |
| `volatile-random` | 从设置了过期时间的 Key 中，随机淘汰 |
| `allkeys-lru` | 从所有 Key 中，淘汰最近最少使用的 |
| `allkeys-random` | 从所有 Key 中，随机淘汰 |
| `no-eviction`（默认） | 内存不足时直接报错，不淘汰任何 Key |

> 重要数据建议设置为永不过期，持久化保存到数据库中。

## 八、总结

本篇覆盖了 Redis 进阶核心知识：

| 模块 | 关键内容 |
|:--|:--|
| 性能基准 | 读 11w/s，写 8.1w/s |
| 持久化 | RDB 快照 / AOF 日志 / 混合模式 |
| 高可用 | 主从复制 → Sentinel → Cluster 三级演进 |
| 缓存问题 | 击穿、穿透、雪崩的识别与解决方案 |
| 淘汰策略 | 6 种策略的适用场景 |

这些知识构成了 Redis 生产应用的基石。结合 [Redis 入门指南](./redis_beginner.md) 中的命令操作，你已经具备了从开发到运维的完整 Redis 知识体系。

## 参考资料

- [Redis 官方文档](https://redis.io/docs/latest/)
- [Redis GitHub 仓库](https://github.com/redis/redis)
