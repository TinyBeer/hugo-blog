---
date: "2026-08-06T11:37:07+08:00"
title: "MySQL -- 实用优化"
tags: ["MySQL", "optimization"]
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

优先级逻辑：索引 > SQL 写法 > 表结构 > 配置参数；80% 性能问题都是索引和 SQL 导致，参数调优收益往往最低<!--more-->  
环境参考：MySQL 8.0，所有示例可直接复制测试

## 收益最大，优先做（解决绝大多数慢查询）

1. 合理建立联合索引，遵循最左前缀原则

   原则：等值条件放最前面，范围条件放索引最后；不要乱建大量单字段索引。  
   ❌ 错误：单独给 user_id、status、create_time 各建一个索引  
   ✅ 正确：(user_id, status, create_time)，status 等值，create_time 范围

   ```sql
   -- 创建联合索引
   CREATE INDEX idx_uid_status_ctime ON t_order(user_id, status, create_time);

   -- 可以命中索引：user_id+status等值，create_time范围
   EXPLAIN SELECT * FROM t_order
   WHERE user_id=10086 AND status=1
   AND create_time >= '2026-01-01' ORDER BY create_time DESC;
   ```

2. 禁止 select \*，只查询需要的字段（覆盖索引）

   减少回表，IO 大幅下降；覆盖索引 Extra 显示 Using index

   ```sql
   -- 坏：select * 回表读取整行
   EXPLAIN SELECT * FROM t_order WHERE user_id=10086 AND status=1;

   -- 好：需要什么查什么，走覆盖索引，不需要回表
   EXPLAIN SELECT id,order_no,create_time FROM t_order
   WHERE user_id=10086 AND status=1;
   ```

3. 避免索引失效常见写法

   不要对索引列做函数运算、隐式转换、like 前缀 %

   ```sql
    -- ❌索引失效，对索引列使用函数
    EXPLAIN SELECT * FROM t_order WHERE DATE(create_time)='2026‑01‑01';
    -- ✅改写，把函数移到常量一侧
    EXPLAIN SELECT * FROM t_order WHERE create_time >= '2026‑01‑01' AND create_time < '2026‑01‑02';

    -- ❌前缀百分号，索引失效
    EXPLAIN SELECT * FROM t_user WHERE phone LIKE '%123456';
    -- ✅后缀百分号可以使用索引
    EXPLAIN SELECT * FROM t_user WHERE phone LIKE '138%';

    -- ❌隐式转换：varchar字段传数字，索引失效
    EXPLAIN SELECT * FROM t_user WHERE phone=13800138000;
    -- ✅字符串匹配字符串
    EXPLAIN SELECT * FROM t_user WHERE phone='13800138000';
   ```

4. 避免 limit 大分页

   ```sql
   -- ❌差
   SELECT * FROM t_order LIMIT 100000,20;

   -- ✅主键id过滤，利用索引定位起点（延迟关联）
   SELECT t.* FROM t_order t
   INNER JOIN (SELECT id FROM t_order ORDER BY id LIMIT 100000,20) AS tmp
   ON t.id=tmp.id;

   -- ✅业务允许优先使用id>offset的方式（性能最优）
   SELECT * FROM t_order WHERE id>100000 LIMIT 20;
   ```

## 高收益，业务开发必守规范

5. join 优化，小表驱动大表

   原则：小表做驱动表，JOIN两边关联字段都建立索引；禁止多表疯狂 join，建议不超过 3 张表。

   ```sql
    -- user(小表) 驱动 order(大表)，order.user_id必须有索引
    EXPLAIN SELECT o.*,u.name
    FROM t_user u
    INNER JOIN t_order o ON u.id=o.user_id
    WHERE u.id IN (1001,1002,1003);
   ```

   > EXPLAIN 看 type，尽量 ref/range，避免 ALL 全表扫描。

6. OR 条件优化，or 左右字段都要有索引；复杂 or 改用 union all

   ```sql
   -- 若 status、user_id各自有索引，可以命中索引合并；没有索引直接全表扫描
   SELECT * FROM t_order WHERE user_id=1001 OR status=2;

   -- ✅等价改写 union all，性能更稳定，避免索引合并的不稳定优化器行为
   SELECT * FROM t_order WHERE user_id=1001
   UNION ALL
   SELECT * FROM t_order WHERE status=2;
   ```

7. 禁止在 where 条件对字段做运算，避免is not null，尽量用默认值代替 null

   > null 无法有效利用索引；尽量设计 NOT NULL，给默认值

   ```sql
    -- ❌不推荐，status允许null，is not null无法高效走索引
    SELECT * FROM t_order WHERE status IS NOT NULL;
    -- ✅设计时：status TINYINT NOT NULL DEFAULT 0
    SELECT * FROM t_order WHERE status !=0;

   ```

## 表结构设计，建表阶段优化，后期改表成本高

8. 字段类型选择，越小越好
   - 时间：优先DATETIME，不要用 varchar 存时间；可选择BIGINT存时间戳
   - 状态：tinyint (1) 代替 int，节省存储空间
   - 字符串：短字符串 char，变长 varchar；大文本不要放在高频查询表，拆分到副表。

9. 主键选择：自增主键优先，避免随机主键（uuid）

   uuid 作为主键会造成页分裂，碎片暴涨，写入性能暴跌

10. 大表分区（千万级以上）

    按时间分区，清理旧数据不需要 delete，直接 drop 分区；适合日志、订单流水表

    ```sql
    CREATE TABLE t_log(
    id BIGINT,
    log_time DATETIME
    )
    PARTITION BY RANGE (TO_DAYS(log_time)) (
    PARTITION p202601 VALUES LESS THAN (TO_DAYS('2026‑02‑01')),
    PARTITION p202602 VALUES LESS THAN (TO_DAYS('2026‑03‑01'))
    );

    ```

## 运维 & 参数优化

> 收益中等，改错容易出问题
> 优先优化 SQL 索引，参数是锦上添花，不是救命稻草

11. 开启慢查询日志，抓慢 SQL

    > 配置分析工具（如：mysqldumpslow） 可更高效的解析日志  
    > 更多详情可以参考 [MySQL -- 慢查询日志](/hugo-blog/note/database/mysql/slow_query_log/)

    ```sql
    -- 查看状态
    SHOW VARIABLES LIKE '%slow_query%';
    SHOW VARIABLES LIKE 'long_query_time';

    -- 会话临时开启，线上配置文件my.cnf永久生效
    SET GLOBAL slow_query_log=ON;
    SET GLOBAL long_query_time=1; --执行超过1秒记录
    ```

12. 核心 InnoDB 关键参数

    ```ini
    # innodb_buffer_pool_size 最重要，物理内存50%‑70%
    innodb_buffer_pool_size=16G
    innodb_log_file_size=2G
    innodb_flush_log_at_trx_commit=1 # 1最高安全；2性能更高丢失风险
    ```

13. 开启死锁日志

    ```ini
      # 开启死锁日志持久化
      innodb_print_all_deadlocks = ON
      # 日志输出到MySQL错误日志文件
      log_error = /var/log/mysql/error.log
    ```

## 参考资料

[MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
