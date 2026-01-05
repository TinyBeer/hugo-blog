---
date: "2026-01-03T16:25:56+08:00"
title: "MySQL -- 基础使用"
tags: ["Database", "MySQL", "SQL"]
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

`MySQL` 是当前最受欢迎、使用最广泛的开源关系型数据库之一。本文中会介绍 `MySQL` 的基础使用。

<!--more-->

## 安装

下面将使用 `docker compose` 安装 `MySQL` 数据库，如果需要直接在宿主机上安装可以参考 [菜鸟教程--MySQL 安装](https://www.runoob.com/mysql/mysql-install.html)

具体步骤如下：

1. 准备好 `compose` 文件

   ```yml
   # compose.yml
   version: "3"
   services:
   mysql:
     container_name: mysql57
     image: mysql:5.7
     command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
     ports:
       - "3306:3306"
     environment:
       - MYSQL_ROOT_PASSWORD=root123
       - MYSQL_USER=dev # 可选：创建的用户名
       - MYSQL_PASSWORD=dev123 # 可选：用户密码（需与 MYSQL_USER 同时设置）
       - TZ=Asia/Shanghai
     volumes:
       - ./log:/var/log
       - ./conf/my.cnf:/etc/my.cnf
       - ./data:/var/lib/mysql
     restart: unless-stopped
   ```

2. 准备一份 `MySQL` 配置文件  
   创建好 `conf/my.cnf` 文件， 并写入配置内容。

   由于直接使用 `docker compose up -d` 启动容器（`conf/my.cnf` 文件不存在），进行卷映射到时候，`docker` 会将 `./conf/my.cnf:/etc/my.cnf` 映射为一个文件夹，而非期望的配置文件，所以这里的配置文件需要提前准备。  
    这里提供两种方式获取配置文件：

   1. 直接使用下方提供的文件

      > 注意确定配置是否兼容

      ```mysql
      # For advice on how to change settings please see
      # http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

      [mysqld]
      #
      # Remove leading # and set to the amount of RAM for the most important data
      # cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
      # innodb_buffer_pool_size = 128M
      #
      # Remove leading # to turn on a very important data integrity option: logging
      # changes to the binary log between backups.
      # log_bin
      #
      # Remove leading # to set options mainly useful for reporting servers.
      # The server defaults are faster for transactions and fast SELECTs.
      # Adjust sizes as needed, experiment to find the optimal values.
      # join_buffer_size = 128M
      # sort_buffer_size = 2M
      # read_rnd_buffer_size = 2M
      skip-host-cache
      skip-name-resolve
      datadir=/var/lib/mysql
      socket=/var/run/mysqld/mysqld.sock
      secure-file-priv=/var/lib/mysql-files
      user=mysql

      # Disabling symbolic-links is recommended to prevent assorted security risks
      symbolic-links=0

      #log-error=/var/log/mysqld.log
      pid-file=/var/run/mysqld/mysqld.pid
      [client]
      socket=/var/run/mysqld/mysqld.sock

      !includedir /etc/mysql/conf.d/
      !includedir /etc/mysql/mysql.conf.d/

      ```

   2. 从所需要使用的镜像中拷贝默认镜像

      ```bash
      # 使用需要使用的 MySQL 镜像启动一个容器
      # 这里容器启动失败也没关系 只要能拷贝出默认配置文件即可
      docker run -d --name mysql-temp mysql:5.7
      # 从容器中拷贝配置文件
      docker cp mysql-temp:/etc/my.cnf .
      # 最后移除容器即可
      docker rm -f mysql-temp
      ```

3. 执行 `docker compose up -d` 命令

## 连接 MySQL

比较常规的方法是使用 `mysql-client` 或者一些第三方工具（如：`Navicat`,`DBeaver`...）连接。  
由于本文使用的镜像中有包含 `mysql-client` 了，这里直接进入容器使用即可，具体命令如下：

```bash
# 1. 进入容器
docker compose exec -it mysql bash
WARN[0000] /home/beer/workspace/deploy/mysql/compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
# 2. 使用 mysql client 连接数据库
bash-4.2# mysql -uroot -p
# 密码为 compose 文件中的 MYSQL_ROOT_PASSWORD 配置： root123
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.44 MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

## MySQL 命令语法描述规则

在介绍基础操作语法前，先对后续会使用到的语法描述符号进行说明。

| 符号                           | 含义                                                          | 举例                                                    |
| :----------------------------- | :------------------------------------------------------------ | :------------------------------------------------------ |
| `[ ]（方括号）`                | 包裹的内容是 可选的（写与不写都不影响语法合法性）             | `[IF NOT EXISTS]`                                       |
| `{ }（大括号）`                | 包裹的内容是 必选的，通常配合 `\|` 使用                       | `{DATABASE \| SCHEMA}`                                  |
| `\|（竖线）`                   | 表示 选择其中一个（互斥选项，只能从多个选项中选择一个）       | `{DATABASE \| SCHEMA}`                                  |
| `...（省略号）`                | 表示前面的语法单元 可以重复多次                               | `[create_option] ...`                                   |
| `UPPERCASE（大写单词）`        | MySQL 关键字（大小写仅作为区分标记，使用时可以小写）          | `CREATE`、` DATABASE`                                   |
| `lowercase（小写单词 / 斜体）` | 用户自定义内容（如 数据库名、字符集名称，需替换为实际业务值） | `db_name`                                               |
| `( )（圆括号）`                | 用于 分组（将多个语法单元视为一个整体，配合 `[]...` 使用）    | `[DEFAULT (CHARACTER SET 字符集 \| COLLATE 排序规则)];` |
| `=（等号）`                    | 表示 赋值 / 指定（可选写，多数场景下省略等号不影响语法生效）  | `ENGINE [=] InnoDB;`                                    |

## 数据库操作

### 创建数据库

语法：

```mysql
CREATE DATABASE [IF NOT EXISTS] db_name [HARACTER SET utf8mb4] [COLLATE utf8mb4_general_ci];
```

解释：

- `[IF NOT EXISITS]` 表示 `IF NOT EXISTS` 是可选的，加上表示只有在不存在名为同名的数据库时才进行创建，这样可以避免重复创建产生的报错
- `db_name` 表示想要创建的数据库名称，需要替换为所需要的数据库名
- `[create_option]` 表示创建参数，主要有两个：
  1. `CHARACTER SET [=] charset_name` 表示需要使用的字符集，常用的是 `utf8mb4` (代表完整的 utf8 字符集， 支持包含各种生僻字的中文、emoji、特殊符号等) 和 `utf8`（阉割版的 utf8 字符集）
  2. `COLLATE [=] collation_name` 表示字符集排序规则，常用的 `utf8mb4_general_ci`,`utf8_general_ci` 与上面的字符集对应，但这个一般比较少主动设置，而是采用默认

示例：

```mysql
CREATE DATABASE mydatabase;

CREATE DATABASE IF NOT EXISTS mydatabase;

CREATE DATABASE mydatabase CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

### 列出数据库

语法：

```mysql
SHOW DATABASES [LIKE 'pattern' | WHERE expr];
```

解释：

通常一个 `MySQL` 中的数据库不会太多，使用 `SHOW DATABASES;` 列出所有数据库可以满足绝大部分使用需求。如果的确需要更精确的查找，可以使用参数设置匹配模式：

- `LIKE 'pattern'` 表示模糊匹配，用于按「通配符规则」模糊查询数据库名，支持两个核心通配符：
  1. `%`：匹配任意长度的任意字符（包括空字符）；
  2. `_`：匹配单个任意字符；
- `WHERE expr` 表示条件匹配，核心用于基于「数据库属性」或「精准名称」查询。

示例：

```mysql
-- 示例1：精准查询名为「mydb」的数据库（仅返回该数据库，无匹配则返回空）
SHOW DATABASES WHERE Schema_name = 'mydb';

-- 示例2：查询名称不以「mysql」开头的数据库（排除系统数据库）
SHOW DATABASES WHERE Schema_name NOT LIKE 'mysql%';

-- 示例3：多条件组合，查询包含「app」且不以「_test」结尾的数据库
SHOW DATABASES
  WHERE Schema_name LIKE '%app%'
  AND Schema_name NOT LIKE '%_test';

-- 示例4：查询数据库名长度大于5的数据库（利用字符串函数，LIKE 无法实现）
SHOW DATABASES WHERE LENGTH(Schema_name) > 5;

```

### 删除数据库

语法：

```mysql
DROP DATABASE [IF EXISTS] db_name;
```

解释：

- `[IF EXISTS]` 表示只有在存在同名数据库时才进行删操作，避免删除不存在的数据库而报错

示例：

```mysql
DROP DATABASE mydatabase;

DROP DATABASE IF EXISTS mydatabase;
```

### 选择数据库

语法：

```mysql
USE db_name;
```

解释:  
在 `MySQL` 中，要选择要使用的数据库，在后续的操作中都会在选中的数据库中执行。

## 表操作

### 创建表

语法：  
由于创建表的完整语法比较复杂，就不进行详细介绍了。这里提供一个基础的语法描述：

```mysql
CREATE TABLE [IF NOT EXISTS] tbl_name
    (create_definition,...)
    [table_options];
```

解释：

- `(create_definition,...)` 主要是字段、索引等信息
- `[table_options]` 是表相关选项，包括引擎、字符集等多设定

示例：

```mysql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    birthdate DATE,
    is_active BOOLEAN DEFAULT TRUE
) ENGINE=InnoDB DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;;
```

- id: 用户 id，整数类型，自增长，作为主键。
- username: 用户名，变长字符串，不允许为空。
- email: 用户邮箱，变长字符串，不允许为空。
- birthdate: 用户的生日，日期类型。
- is_active: 用户是否已经激活，布尔类型，默认值为 true。
- ENGINE： 设置存储引擎为 InnoDB
- CHARSET： 设置编码为 utf8mb4
- COLLATE： 设置字符集排序方式

示例中仅使用了部分数据类型，更全面的数据类型说明可以参考 [数据类型](#数据类型)

### 列出表

语法：

```mysql
SHOW TABLES
    [{FROM | IN} db_name]
    [LIKE 'pattern' | WHERE expr]
```

解释：

- `[{FROM | IN} db_name]` 指明那个数据库，如果不指明则列出当前数据库的表信息
- 其他语法可以类比 [列出数据库](#列出数据库)

### 删除表

语法：

```yaml
DROP TABLE [IF EXISTS]
tbl_name [, tbl_name] ...;
```

解释：

- `[IF EXISTS]` 如果表存在才执行删除操作，避免由于表不存在而发生报错。
- `tbl_name [, tbl_name] ...` 同时删除多长表时，使用 `,` 分隔表名。

示例：

```mysql
-- 删除表，如果存在的话
DROP TABLE IF EXISTS mytable;

-- 直接删除表，不检查是否存在
DROP TABLE mytable, mytalbe2;

```

## 数据操作

### 插入数据

语法：

```mysql
INSERT INTO tbl_name
    [(col_name [, col_name] ...)]
    VALUES  (value_list) [, (value_list)] ...;
```

解释：

- `[(col_name [, col_name] ...)]` 插入数据时如果不希望为所有列指定值，则需指定哪些列赋指定值。
- ` (value_list) [, (value_list)] ...` 要插入的每一条数据中指定的列值，使用 `()` 进行分组，如果一次需要插入多条数据，使用 `,` 分隔。
  示例：

```mysql
-- 单条数据插入 仅插入指定列 --
INSERT INTO users (username, email, birthdate, is_active) VALUES ('test', 'test@runoob.com', '1990-01-01', true);

-- 单条数据插入 插入所有列时，省略列名 --
INSERT INTO users VALUES ('test', 'test@runoob.com', '1990-01-01', true);

-- 批量数据插入 --
-- 插入所有列也可以省略列名--
INSERT INTO users (username, email, birthdate, is_active)
VALUES
    ('test1', 'test1@runoob.com', '1985-07-10', true),
    ('test2', 'test2@runoob.com', '1988-11-25', false),
    ('test3', 'test3@runoob.com', '1993-05-03', true);
```

### 查询数据

对于数据库的查询，有一套非常复杂的语法机制，这里只介绍最常用的条件查询。当然，示例中会列出了一些常用定式。

语法：

```mysql
SELECT {* | (col_name [, col_name] ...)}
    FROM table_name
    [WHERE condition]
    [ORDER BY col_name [ASC | DESC]]
    [LIMIT number];
```

解释：

- `{* | (col_name [, col_name] ...)}` 指定返回哪些列，如果需要返回所有列则使用 `*`
- `[WHERE condition]` 筛选满足条件的数据，常用的条件格式为 `col_name = value [AND col_name = value] ...`
- `[ORDER BY col_name [ASC | DESC]]` 数据排序规则，依据哪些字段排序，排序规则是升序还是降序。
- `[LIMIT number]` 限制返回值数量。

示例：

```mysql
-- 选择所有列的所有行
SELECT * FROM users;

-- 选择特定列的所有行
SELECT username, email FROM users;

-- 添加 WHERE 子句，选择满足条件的行
SELECT * FROM users WHERE is_active = TRUE;

-- 添加 ORDER BY 子句，按照某列的升序排序
SELECT * FROM users ORDER BY birthdate;

-- 添加 ORDER BY 子句，按照某列的降序排序
SELECT * FROM users ORDER BY birthdate DESC;

-- 添加 LIMIT 子句，限制返回的行数
SELECT * FROM users LIMIT 10;

-- 使用 AND 运算符和通配符
SELECT * FROM users WHERE username LIKE 'j%' AND is_active = TRUE;

-- 使用 OR 运算符
SELECT * FROM users WHERE is_active = TRUE OR birthdate < '1990-01-01';

-- 使用 IN 子句
SELECT * FROM users WHERE birthdate IN ('1990-01-01', '1992-03-15', '1993-05-03');
```

### 删除数据

语法：

```mysql
DELETE FROM tbl_name
    [WHERE where_condition]
```

解释：

删除数据在生产环境不是一个常用的操作，相比于直接删除数据，我们会更多的使用软删除。  
即便是需要删除数据 `DELETE FROM tbl_name [WHERE where_condition]` 也可以满足大部分需求。

示例：

```mysql
-- 删除 id 为 123456 的用户表数据
DELETE FROM user WHERE id = 123456;
```

### 更新数据

语法：

```mysql
UPDATE table_name
    SET col_name = value [, vol_name = value] ...
    [WHERE where_condition]
```

解释：  


- `table_name` 是你要更新数据的表的名称。
- `col_name = value [, vol_name = value]...` 是你要更新的列以及要改为的值。
- `WHERE condition` 是一个可选的子句，用于指定更新的行。如果省略 `WHERE `子句，将更新表中的所有行。

示例：

```mysql
-- 更新单个列的值
UPDATE employees SET salary = 60000 WHERE employee_id = 101;

-- 更新多个列的值
UPDATE orders SET status = 'Shipped', ship_date = '2023-03-01' WHERE order_id = 1001;

-- 使用表达式更新值
UPDATE products SET price = price * 1.1 WHERE category = 'Electronics';
```

## 数据类型

`MySQL` 支持多种类型，大致可以分为三类：`数值`、`日期/时间`和 `字符串(字符)` 类型。

数值类型：

| 类型           | 大小 （bytes）                                | 范围（有符号）                                                                                                                      | 范围（无符号）                                                    | 用途           |
| :------------- | :-------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------- | :------------- |
| TINYINT        | 1                                             | (-128，127)                                                                                                                         | (0，255)                                                          | 小整数值       |
| SMALLINT       | 2                                             | (-32 768，32 767)                                                                                                                   | (0，65 535)                                                       | 大整数值       |
| MEDIUMINT      | 3                                             | (-8 388 608，8 388 607)                                                                                                             | (0，16 777 215)                                                   | 大整数值       |
| INT 或 INTEGER | 4                                             | (-2 147 483 648，2 147 483 647)                                                                                                     | (0，4 294 967 295)                                                | 大整数值       |
| BIGINT         | 8                                             | (-9 223 372 036 854 775 808，9 223 372 036 854 775 807)                                                                             | (0，18 446 744 073 709 551 615)                                   | 极大整数值     |
| FLOAT          | 4                                             | (-3.402 823 466 E+38，-1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38)                                         | 0，(1.175 494 351 E-38，3.402 823 466 E+38)                       | 单精度浮点数值 |
| DOUBLE         | 8                                             | (-1.797 693 134 862 315 7 E+308，-2.225 073 858 507 201 4 E-308)，0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308) | 0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308) | 双精度浮点数值 |
| DECIMAL        | 对 DECIMAL(M,D) ，如果 M>D，为 M+2 否则为 D+2 | 依赖于 M 和 D 的值                                                                                                                  | 依赖于 M 和 D 的值                                                | 小数值         |

日期/时间类型：

| 类型      | 大小(bytes) | 范围                                                   | 格式                | 用途                     |
| :-------- | :---------- | :----------------------------------------------------- | :------------------ | :----------------------- |
| DATE      | 3           | 1000-01-01/9999-12-31                                  | YYYY-MM-DD          | 日期值                   |
| TIME      | 3           | '-838:59:59'/'838:59:59'                               | HH:MM:SS            | 时间值或持续时间         |
| YEAR      | 1           | 1901/2155                                              | YYYY                | 年份值                   |
| DATETIME  | 8           | '1000-01-01 00:00:00' 到 '9999-12-31 23:59:59'         | YYYY-MM-DD hh:mm:ss | 混合日期和时间值         |
| TIMESTAMP | 4           | '1970-01-01 00:00:01' UTC 到 '2038-01-19 03:14:07' UTC | YYYY-MM-DD hh:mm:ss | 混合日期和时间值，时间戳 |

字符串/字符类型：

| 类型       | 大小(bytes)     | 用途                            |
| :--------- | :-------------- | :------------------------------ |
| CHAR       | 0-255           | 定长字符串                      |
| VARCHAR    | 0-65535         | 变长字符串                      |
| TINYBLOB   | 0-255           | 不超过 255 个字符的二进制字符串 |
| TINYTEXT   | 0-255           | 短文本字符串                    |
| BLOB       | 0-65 535        | 二进制形式的长文本数据          |
| TEXT       | 0-65 535        | 长文本数据                      |
| MEDIUMBLOB | 0-16 777 215    | 二进制形式的中等长度文本数据    |
| MEDIUMTEXT | 0-16 777 215    | 中等长度文本数据                |
| LONGBLOB   | 0-4 294 967 295 | 二进制形式的极大文本数据        |
| LONGTEXT   | 0-4 294 967 295 | 极大文本数据                    |

> 注意：char(n) 和 varchar(n) 中括号中 n 代表字符的个数，并不代表字节个数，比如 CHAR(30) 就可以存储 30 个字符。

## 参考资料

[MySQL 官方文档](https://dev.mysql.com/)  
[菜鸟教程 MySQL](https://www.runoob.com/mysql)
