---
date: "2026-01-26T10:59:39+08:00"
title: "MySQL -- 慢查询日志"
tags: ["MySQL", "Optimization"]
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

`MySQL` 慢查询日志是用于记录执行时间超过指定阈值的 `SQL` 语句的日志功能，是优化数据库性能的核心工具之一。

<!--more-->

慢查询日志会记录执行时长超过阈值（`long_query_time`）并且 扫描行数不少于配置值（`min_examined_row_limit`）的 `SQL` 语句，可以帮助开发者快速定位到执行效率低下的 `SQL` 语句。

> 开启慢查询日志不可避免的会产生额外的性能开销（主要是记录日志的 I/O 开销），生产环境中建议仅在需要时开启，并及时关闭。

## 常用参数

- `slow_query_log`  
   慢查询日志开关，设置 `1` 为开启、`0` 为关闭。操作方法如下：

  ```mysql
  -- 查看慢查询日志是否开启
  SHOW VARIABLES LIKE 'slow_query_log';
  -- 临时开启慢查询日志（1=开启，0=关闭）也可以使用（ON/OFF）
  SET GLOBAL slow_query_log = 1;
  -- 使用完后关闭慢查询日志
  SET GLOBAL slow_query_log = 0;
  ```

- `slow_query_log_file`  
   查询日志存储路径，一般不需要手动设置，使用方法如下：

  ```mysql
  -- 查看慢查询日志存储路径（如果需要查看日志文件）
  SHOW VARIABLES LIKE 'slow_query_log_file';
  -- 指定慢查询日志存储路径
  SET GLOBAL slow_query_log_file = '/var/lib/mysql/slow_query.log';
  ```

- `long_query_time`  
   慢查询日志阈值，单位秒，默认值为 10,使用方法如下：

  ```mysql
  -- 查看慢查询阈值（单位：秒）
  SHOW VARIABLES LIKE 'long_query_time';
  -- 临时设置慢查询阈值为3秒（对新连接立即生效，当前连接需重新连接或执行下面的会话级设置）
  SET GLOBAL long_query_time = 3;
  -- 让当前会话立即生效3秒阈值（可选，避免重新连接）
  SET SESSION long_query_time = 3;
  ```

- `log_queries_not_using_indexes`  
   是否记录未使用所以的语句，默认不记录，使用方法如下：

  ```mysql
    -- 查看无索引日志记录开关
    SHOW VARIABLES LIKE 'log_queries_not_using_indexes';
    -- 开启无索引日志记录开关
    SET GLOBAL log_queries_not_using_indexes = 1;
    -- 关闭无索引查询记录
    SET GLOBAL log_queries_not_using_indexes = 0;
  ```

此外还有 `min_examined_row_limit`、`log_slow_admin_statements`等参数，由于使用较少这里就不进行说明了，详情可参考[官方文档](https://dev.mysql.com/doc/refman/5.7/en/slow-query-log.html)。

## 示例

### 临时开启

> [!important] 临时开启慢查询日志，会在 `MySQL` 重启后失效。

逐条执行以下命令：

```mysql
SET GLOBAL slow_query_log = 1; -- 开启慢查询日志

SET SESSION long_query_time = 3; -- 执行时间超过 3 秒的 SQL 会被记录

SET GLOBAL log_queries_not_using_indexes = 1; -- 记录未使用索引的 SQL

SHOW VARIABLES LIKE 'slow_query_log_file'; -- 查看慢查询日志存储路径

SET GLOBAL slow_query_log = 0; -- 开启慢查询日志


SELECT SLEEP(4); -- 执行一条慢查询语句

-- 恢复
SET GLOBAL slow_query_log = 0; -- 关闭慢查询日志

SET GLOBAL log_queries_not_using_indexes = 0; -- 不记录未使用索引的 SQL
```

执行完成后，我们找到慢查询日志文件，可以看到其中多了一条类似如下格式的日志记录：

```plaintext
# Time: 2026-01-26T02:15:12.201909Z
# User@Host: dev[dev] @  [192.168.1.58]  Id: 6568855
# Query_time: 4.000240  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 1
SET timestamp=1769393708;
select sleep(4);
```

### 日志说明

其中记录了日志记录时间（`Time`）、执行者信息（`User@Host: dev[dev] @  [192.168.1.58]  Id: 6568855`）、查询时间（`Query_time`）、锁等待时间（`Lock_time`），返回行数（`Rows_sent`），扫描行数（`Rows_examined`）、执行开始时间戳（`SET timestamp=1769393708;`）、执行的 SQL 语句信息。

## 配置文件

如果希望重启 `MySQL` 服务后仍然开启慢查询日志（一般是在测试环境中），可以通过在配置文件中添加一下配置实现：

```ini
[mysqld]
# 开启慢查询日志
slow_query_log = ON
# 日志文件路径
slow_query_log_file = /var/lib/mysql/slow_query.log
# 慢查询阈值（秒）
long_query_time = 3
# 记录未使用索引的 SQL
log_queries_not_using_indexes = ON
# 记录管理员慢操作
log_slow_admin_statements = ON
# 可选：日志输出格式（FILE 写入文件，TABLE 写入 mysql.slow_log 表，默认 FILE）
log_output = FILE
```

## 日志工具

通过一段时间的慢查询日志记录，我们获得到大量的慢查询记录，这时从一对记录中找到需要的就会成为一项耗时的工作。好在，`MySQL` 提供了配套的查询工具（`mysqldumpslow`）。用法如下：

```bash
mysqldumpslow [options] [log_file ...]
```

选项：

| 选项        | 作用                                               |
| :---------- | :------------------------------------------------- |
| `-a`        | 不把所有数字抽象成 `N`，字符串抽象成 `'S'`         |
| `-n`        | 仅替换位数≥指定值的数字为 `N`                      |
| `--debug`   | 输出调试信息                                       |
| `-g`        | 只分析匹配指定模式的 SQL 语句                      |
| `--help`    | 显示帮助信息并退出                                 |
| `-h`        | 指定日志文件名中的服务器主机名                     |
| `-i`        | 指定服务器实例名                                   |
| `-l`        | 不把锁等待时间从总执行时间中扣除                   |
| `-r`        | 反转排序顺序，默认是 “从慢到快”“从多到少”          |
| `-s`        | 指定输出结果的排序规则                             |
| `-t`        | 只显示前 `N` 条慢查询                              |
| `--verbose` | 详细模式，会增加执行次数、平均执行时间、锁定时间等 |

其中 `-s` 常用取值如下：

- `t`：按 `Query_time`（执行时间）排序（默认）；
- `c`：按执行次数排序；
- `r`：按 `Rows_examined`（扫描行数）排序；
- `at`：按平均执行时间排序；
- `ar`：按平均扫描行数排序。

### 常用模式

```bash
# 1. 查看执行时间最长的 10 条原始 SQL（保留具体值）
mysqldumpslow -a -s t -t 10 /var/lib/mysql/slow_query.log

# 2. 查看 users 表中执行次数最多的 5 条慢查询（详细模式）
# -g "users" 匹配语句中包含 users 的记录
mysqldumpslow --verbose -g "users" -s c -t 5 /var/lib/mysql/slow_query.log

# 3. 查看扫描行数最多的 10 条慢查询（含锁等待时间）
mysqldumpslow -l -s r -t 10 /var/lib/mysql/slow_query.log

```

## 参考资料

[MySQL 官方文档](https://dev.mysql.com/)
