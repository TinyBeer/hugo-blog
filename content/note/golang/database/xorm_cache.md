---
date: "2025-12-27T16:31:27+08:00"
title: "XORM -- 查询缓存"
tags: ["Golang", "XORM", "Database", "ORM", "Cache"]
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

`XORM` 内置了一致性缓存支持，不过默认并没有开启。要开启缓存，需要在 `engine` 创建完后进行配置。

<!--more-->

## 开启本地缓存

### 启用一个全局的内存缓存

```golang
cacher := caches.NewLRUCacher(caches.NewMemoryStore(), 1000)
engine.SetDefaultCacher(cacher)
```

上述代码采用了 `LRU` 算法的一个缓存，缓存方式是存放到内存中，缓存 `struct` 的记录数为 `1000` 条，缓存针对的范围是所有具有主键的表，没有主键的表中的数据将不会被缓存。

### 为指定表开启缓存

```golang
cacher := caches.NewLRUCacher(caches.NewMemoryStore(), 1000)
engine.MapCacher(&user, cacher)
```

### 禁用某个表的缓存

```golang
engine.MapCacher(&user, nil)
```

## 注意!!!

在使用 `XORM` 的缓存功能时候，需要注意以下几点：

1. 当使用了 Distinct,Having,GroupBy 方法将不会使用缓存

2. 在 Get 或者 Find 时使用了 Cols,Omit 方法，则在开启缓存后此方法无效，系统仍旧会取出这个表中的所有字段。

3. 在使用 Exec 方法执行了方法之后，可能会导致缓存与数据库不一致的地方。因此如果启用缓存，尽量避免使用 Exec。如果必须使用，则需要在使用了 Exec 之后调用 ClearCache 手动做缓存清除的工作。比如：
   ```golang
   engine.Exec("update user set name = ? where id = ?", "xlw", 1)
   engine.ClearCache(new(User))
   ```

## 验证缓存生效方法

验证过程均针对一下代码进行:

```golang
type User struct {
	Id      int64
	Name    string
	Salt    string
	Age     int
	Passwd  string    `xorm:"varchar(200)"`
	Created time.Time `xorm:"created"`
	Updated time.Time `xorm:"updated"`
}

...
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
...
```

### 开启 MySQL 通用查询日志

```sql
-- 1. 查看通用查询日志是否开启
mysql> SHOW VARIABLES LIKE 'general_log';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| general_log   | OFF   |
+---------------+-------+
1 row in set (0.01 sec)

-- 2. 查看通用查询日志文件
mysql> SHOW VARIABLES LIKE 'general_log_file';
+------------------+---------------------------------+
| Variable_name    | Value                           |
+------------------+---------------------------------+
| general_log_file | /var/lib/mysql/3de1f99389cc.log |
+------------------+---------------------------------+
1 row in set (0.00 sec)

-- 3. 临时开启
SET GLOBAL general_log = ON;

-- 4. 临时关闭
SET GLOBAL general_log = OFF;
```

此时分别在开启和关闭缓存的情况下查询,得到如下日志:

```bash
# 未开启缓存
2025-12-27T08:30:00.952183Z	  23 Connect	root@172.18.0.1 on test using TCP/IP
2025-12-27T08:30:00.952284Z	  23 Query	SET NAMES utf8
2025-12-27T08:30:00.952426Z	  23 Prepare	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1
2025-12-27T08:30:00.952448Z	  23 Execute	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=1 LIMIT 1
2025-12-27T08:30:00.952649Z	  23 Close stmt
2025-12-27T08:30:00.952733Z	  23 Prepare	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1
2025-12-27T08:30:00.952756Z	  23 Execute	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=1 LIMIT 1
2025-12-27T08:30:00.952812Z	  23 Close stmt
2025-12-27T08:30:00.952857Z	  23 Prepare	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE (id = ?) LIMIT 1
2025-12-27T08:30:00.952876Z	  23 Execute	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE (id = 1) LIMIT 1
2025-12-27T08:30:00.952925Z	  23 Close stmt


# 开启缓存
2025-12-27T08:43:37.003673Z	  24 Connect	root@172.18.0.1 on test using TCP/IP
2025-12-27T08:43:37.003796Z	  24 Query	SET NAMES utf8
2025-12-27T08:43:37.003917Z	  24 Prepare	SELECT `id` FROM `user` WHERE `id`=? LIMIT 1
2025-12-27T08:43:37.003935Z	  24 Execute	SELECT `id` FROM `user` WHERE `id`=1 LIMIT 1
2025-12-27T08:43:37.004331Z	  25 Connect	root@172.18.0.1 on test using TCP/IP
2025-12-27T08:43:37.004396Z	  25 Query	SET NAMES utf8
2025-12-27T08:43:37.004477Z	  25 Prepare	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1
2025-12-27T08:43:37.004506Z	  25 Execute	SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=1 LIMIT 1
2025-12-27T08:43:37.004658Z	  25 Close stmt
2025-12-27T08:43:37.004684Z	  24 Close stmt
2025-12-27T08:43:37.004808Z	  24 Prepare	SELECT `id` FROM `user` WHERE (id = ?) LIMIT 1
2025-12-27T08:43:37.004826Z	  24 Execute	SELECT `id` FROM `user` WHERE (id = 1) LIMIT 1
2025-12-27T08:43:37.004896Z	  24 Close stmt
```

可以看到开启缓存后, `XORM` 减少了对数据库的访问。

### 利用 XORM 事件

为 `User` 结构体注册 `AfterLoad` 事件,该事件在 `Get` 或 `Find` 方法中，当数据已经从数据库查询出来，而在设置到结构体之后调用。

```golang
func (u *User) AfterLoad(session *xorm.Session) {
	log.Printf("load user: %v", u)
}
```

执行相同代码得到如下日志输出:

```bash
# 为开启缓存
[xorm] [info]  2025/12/27 16:30:00.952545 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1 [1] - 855.801µs
2025/12/27 16:30:00 load user: &{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
[xorm] [info]  2025/12/27 16:30:00.952789 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1 [1] - 107.682µs
2025/12/27 16:30:00 load user: &{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
[xorm] [info]  2025/12/27 16:30:00.952903 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE (id = ?) LIMIT 1 [1] - 76.758µs
2025/12/27 16:30:00 load user: &{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}

# 开启缓存
[xorm] [info]  2025/12/27 16:43:37.004019 [SQL] SELECT `id` FROM `user` WHERE `id`=? LIMIT 1 [1] - 788.641µs
[xorm] [info]  2025/12/27 16:43:37.004598 [SQL] SELECT `id`, `name`, `salt`, `age`, `passwd`, `created`, `updated` FROM `user` WHERE `id`=? LIMIT 1 [1] - 481.313µs
2025/12/27 16:43:37 load user: &{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
[xorm] [info]  2025/12/27 16:43:37.004874 [SQL] SELECT `id` FROM `user` WHERE (id = ?) LIMIT 1 [1] - 124.513µs
true <nil>
&{1 tom salt 18 123456 2025-12-26 21:05:43 +0800 CST 2025-12-26 21:05:43 +0800 CST}
```

可以看到开启本地缓存后，`XORM` 仅触发了一次 `AfterLoad` 事件。

### 其他

此外，还可以通过加载缓存后手动修改数据库数据进行验证。

## 使用 Redis 进行缓存

### 为指定对象实现缓存方案

当前实现了内存存储的 `CacheStore` 接口 `MemoryStore` ，如果需要采用其它设备存储，可以实现 `CacheStore` 接口。

这里简单演示以下使用 `Redis` 缓存 `User` 数据：

1. 使用 `Redis` 实现 `CacheStore` 接口

   ```golang
   type RedisStore struct {
   	client *redis.Client
   }

   func NewRedisStore() *RedisStore {
   	client := redis.NewClient(&redis.Options{
   		Addr:     "localhost:6379",
   		Password: "",
   		DB:       0,
   		PoolSize: 20,
   	})
   	fmt.Println(client.Ping(context.Background()).Result())
   	return &RedisStore{
   		client: client,
   	}
   }

   // Del implements caches.CacheStore.
   func (r *RedisStore) Del(key string) error {
   	return r.client.Del(context.TODO(), key).Err()
   }

   // Get implements caches.CacheStore.
   func (r *RedisStore) Get(key string) (interface{}, error) {
   	str, err := r.client.Get(context.TODO(), key).Result()
   	if err != nil {
   		return nil, err
   	}

   	u := new(User)
   	err = json.Unmarshal([]byte(str), u)
   	return u, err
   }

   // Put implements caches.CacheStore.
   func (r *RedisStore) Put(key string, value interface{}) error {
   	bs, _ := json.Marshal(value)
   	return r.client.Set(context.TODO(), key, string(bs), 0).Err()
   }

   var _ caches.CacheStore = (*RedisStore)(nil)
   ```

2. 将本地缓存切换为刚才实现的 `CacheStore` `Redis` 缓存

   ```golang
   	cacher := caches.NewLRUCacher(NewRedisStore(), 1000)
   ```

3. 运行之前的代码后查看 `Redis` 中的数据

   ```plaintext
   127.0.0.1:6379> keys *
   1) "SELECT `id` FROM `user` WHERE `id`=? LIMIT 1-[1]"
   2) "SELECT `id` FROM `user` WHERE (id = ?) LIMIT 1-[1]"
   3) "user-\x0f\x7f\x02\x01\x01\x02PK\x01\xff\x80\x00\x01\x10\x00\x00\x0e\xff\x80\x00\x01\x05int64\x04\x02\x00\x02"
   127.0.0.1:6379> get "SELECT `id` FROM `user` WHERE `id`=? LIMIT 1-[1]"
   "\"\\r\\ufffd\\ufffd\\u0002\\u0001\\u0002\\ufffd\\ufffd\\u0000\\u0001\\ufffd\\ufffd\\u0000\\u0000\\u000f\x7f\\u0002\\u0001\\u0001\\u0002PK\\u0001\\ufffd\\ufffd\\u0000\\u0001\\u0010\\u0000\\u0000\\u000f\\ufffd\\ufffd\\u0000\\u0001\\u0001\\u0005int64\\u0004\\u0002\\u0000\\u0002\""
   127.0.0.1:6379> get "user-\x0f\x7f\x02\x01\x01\x02PK\x01\xff\x80\x00\x01\x10\x00\x00\x0e\xff\x80\x00\x01\x05int64\x04\x02\x00\x02"
   "{\"Id\":1,\"Name\":\"tom\",\"Salt\":\"salt\",\"Age\":18,\"Passwd\":\"123456\",\"Created\":\"2025-12-26T21:05:43+08:00\",\"Updated\":\"2025-12-26T21:05:43+08:00\"}"
   127.0.0.1:6379>

   ```

   可以看到缓存数据已经倒入的 `Redis` 中了。

### 社区中的方案

由于 `CacheStore` 接口中 `Get(key string) (interface{}, error)` 没有将返回数据的结构传入，无法实现通用的方案。
于是社区中给出了另一种解决思路： `xorm-redis-cache`，直接实现 `Redis` 缓存 `Cacher`。

> 安装 `xorm-redis-cache`
> go get github.com/go-xorm/xorm-redis-cache

```golang
// // New a Redis Cacher, host as IP endpoint, i.e., localhost:6379, provide empty string or nil if Redis server doesn't
// require AUTH command, defaultExpiration sets the expire duration for a key to live. Until redigo supports
// sharding/clustering, only one host will be in hostList
//
//     engine.SetDefaultCacher(xormrediscache.NewRedisCacher("localhost:6379", "", xormrediscache.DEFAULT_EXPIRATION, engine.Logger))
//
// or set MapCacher
//
//     engine.MapCacher(&user, xormrediscache.NewRedisCacher("localhost:6379", "", xormrediscache.DEFAULT_EXPIRATION, engine.Logger))
//
// func NewRedisCacher(host string, password string, defaultExpiration time.Duration, logger core.ILogger) *RedisCacher

// 使用本地redis  无密码  过期时间为 30s
engine.SetDefaultCacher(xormRedisCache.NewRedisCacher("localhost:6379", "", time.Second*30, engine.Logger()))
```

## 参考资料

[Xorm 官方文档](https://xorm.io/)  
[MySQL 官方文档](https://dev.mysql.com/)  
[xorm-redis-cache](https://github.com/go-xorm/xorm-redis-cache)
