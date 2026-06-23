---
title: Redis 基础入门
date: 2026-06-23 18:30:00
tags: Redis, 数据库, 缓存
categories: Posts
excerpt: Redis 核心概念、五大基础数据类型、常用命令速查、持久化机制与应用场景
---

## 什么是 Redis？

Redis（**Re**mote **Di**ctionary **S**erver）是一个开源的**内存数据结构存储系统**，可以用作：

- **数据库** — 持久化存储数据
- **缓存** — 加速读取热点数据
- **消息队列** — 发布/订阅、流处理

核心特点：**所有数据存在内存中，读写速度极快**（单机 10W+ QPS）。

---

## 为什么快？

| 原因 | 说明 |
|------|------|
| 内存操作 | 数据在 RAM 中，无需磁盘 IO |
| 单线程模型 | 无锁竞争，避免上下文切换开销 |
| IO 多路复用 | epoll 高效处理大量并发连接 |
| 高效数据结构 | SDS、跳表、压缩列表等优化实现 |

> Redis 6.0 开始引入多线程 IO，但**命令执行仍是单线程**。

---

## 五大基础数据类型

### 1. String（字符串）

最基础的类型，一个 key 对应一个 value，最大 **512MB**。

```bash
SET user:1:name "Jinn"
GET user:1:name          # "Jinn"
SET counter 1
INCR counter              # 2（原子自增）
```

**典型场景**：缓存用户信息、计数器、分布式锁（`SETNX`）、Session。

### 2. Hash（哈希）

一个 key 下面有多个 field-value 对，类似 Go 的 `map[string]string`。

```bash
HSET user:1 name "Jinn" age 25 role "backend"
HGET user:1 name           # "Jinn"
HGETALL user:1
```

**典型场景**：对象存储（用户信息、商品详情），比 String 省**序列化开销**。

### 3. List（列表）

有序字符串链表，支持两端插入弹出。

```bash
RPUSH queue:tasks "task1" "task2" "task3"
LPUSH queue:tasks "task0"
LPOP queue:tasks           # "task0"（先进先出 = 队列）
RPOP queue:tasks           # "task3"（后进先出 = 栈）
```

**典型场景**：消息队列、最新消息列表、朋友圈时间线。

### 4. Set（无序集合）

不重复元素集合，支持交集/并集/差集运算。

```bash
SADD tags:golang "goroutine" "channel" "interface"
SADD tags:python "async" "generator" "channel"
SINTER tags:golang tags:python   # {"channel"} 交集
```

**典型场景**：标签系统、共同好友、抽奖（`SRANDMEMBER`）、去重。

### 5. ZSet（有序集合）

每个元素关联一个 `score`，按分数排序。底层用**跳表**实现。

```bash
ZADD leaderboard 100 "Jinn" 95 "Alice" 88 "Bob"
ZREVRANGE leaderboard 0 2 WITHSCORES
ZINCRBY leaderboard 5 "Bob"    # Bob: 88 → 93
```

**典型场景**：排行榜、延迟队列、滑动窗口限流。

---

## 数据类型选择速查

| 场景 | 推荐类型 |
|------|---------|
| 简单 KV 缓存 | String |
| 对象属性存储 | Hash |
| 队列 / 最新列表 | List |
| 去重 / 标签 | Set |
| 排行榜 / 排序 | ZSet |

---

## 基础命令速查

### 🔑 通用命令

```bash
KEYS pattern          # 查找所有匹配的 key（慎用！生产环境会阻塞）
KEYS user:*            # 例：返回 user:1, user:2...
EXISTS key             # 判断 key 是否存在，返回 1/0
TYPE key               # 查看 key 的数据类型
DEL key1 key2          # 删除一个或多个 key
EXPIRE key 60          # 设置 60 秒后过期
TTL key                # 查看剩余过期时间（-1=永不过期，-2=已过期）
RENAME key newkey      # 重命名
SCAN cursor [MATCH pattern] [COUNT count]  # 增量式遍历 key（替代 KEYS）
```

### 📝 String 命令

```bash
SET key value [EX seconds] [NX|XX]    # 设置值
GET key                                # 获取值
SET key value NX                       # 只在 key 不存在时设置
SET key value XX                       # 只在 key 已存在时设置
MSET k1 v1 k2 v2                       # 批量设置
MGET k1 k2                             # 批量获取
INCR key                               # 原子 +1
INCRBY key 10                          # 原子 +10
DECR key                               # 原子 -1
INCRBYFLOAT key 2.5                    # 浮点数递增
APPEND key "world"                     # 追加字符串
STRLEN key                             # 字符串长度
GETRANGE key 0 4                       # 截取子串 [0,4]
SETBIT key 7 1                         # 按位操作
BITCOUNT key                           # 统计值为 1 的位数
```

### 🗂️ Hash 命令

```bash
HSET key field value                   # 设置 field
HGET key field                          # 获取 field
HMSET key f1 v1 f2 v2                   # 批量设置
HMGET key f1 f2                         # 批量获取
HGETALL key                             # 获取所有 field-value
HDEL key field1 field2                  # 删除 field
HEXISTS key field                       # field 是否存在
HKEYS key                               # 获取所有 field
HVALS key                               # 获取所有 value
HLEN key                                # field 数量
HINCRBY key field 5                     # field 值 +5
```

### 📋 List 命令

```bash
LPUSH key v1 v2                         # 从左侧压入（头部）
RPUSH key v1 v2                         # 从右侧压入（尾部）
LPOP key                                # 从左侧弹出
RPOP key                                # 从右侧弹出
LRANGE key 0 -1                         # 获取范围（0=-1 = 全部）
LLEN key                                # 列表长度
LINDEX key 2                            # 按索引获取（0-based）
LINSERT key BEFORE|AFTER "pivot" "val"  # 在某元素前/后插入
LREM key 2 "val"                        # 删除前 N 个匹配值
LTRIM key 0 99                          # 只保留指定范围（裁剪）
```

> LPUSH + RPOP = **队列（先进先出）**
> LPUSH + LPOP = **栈（后进先出）**

### 🎯 Set 命令

```bash
SADD key member1 member2                # 添加成员
SMEMBERS key                            # 获取所有成员
SISMEMBER key member                    # 判断成员是否存在
SREM key member                         # 删除成员
SCARD key                               # 成员数量
SRANDMEMBER key 3                       # 随机获取 3 个（不移除）
SPOP key 3                              # 随机弹出 3 个（移除）
SDIFF key1 key2                         # 差集（key1 有但 key2 没有）
SINTER key1 key2                        # 交集
SUNION key1 key2                        # 并集
SDIFFSTORE dest key1 key2              # 差集结果存到 dest
SINTERSTORE dest key1 key2             # 交集结果存到 dest
```

### 🏆 ZSet（有序集合）命令

```bash
ZADD key score1 member1 score2 member2  # 添加（带分数）
ZRANGE key 0 -1                         # 按分数升序获取
ZRANGE key 0 -1 WITHSCORES              # 带分数输出
ZREVRANGE key 0 -1                      # 按分数降序获取
ZSCORE key member                       # 获取某个成员的分数
ZRANK key member                        # 升序排名（0-based）
ZREVRANK key member                     # 降序排名
ZINCRBY key 5 member                    # 分数 +5
ZREM key member                         # 删除成员
ZCARD key                               # 成员总数
ZCOUNT key 60 100                       # 分数在 60-100 之间的成员数
ZRANGEBYSCORE key 60 100                # 获取分数在 60-100 之间的成员
ZREMRANGEBYRANK key 0 2                 # 按排名删除前 3 名
```

### 📊 服务器与运维命令

```bash
INFO                    # 服务器详细信息（内存、连接、命中率等）
INFO memory             # 只看内存信息
DBSIZE                  # 当前数据库 key 总数
FLUSHDB                 # 清空当前数据库
FLUSHALL                # 清空所有数据库（危险！）
CLIENT LIST             # 查看所有客户端连接
CLIENT KILL id          # 踢掉某个客户端
CONFIG GET maxmemory    # 查看配置项
CONFIG SET maxmemory 1gb  # 动态修改配置
MEMORY USAGE key        # key 占用的内存字节数
SLOWLOG GET 10          # 查看最近 10 条慢查询
```

### ⏱️ Key 过期命令

```bash
EXPIRE key 300                # 300 秒后过期
EXPIREAT key 1719000000       # 在指定时间戳过期
PERSIST key                   # 取消过期（变为永久）
TTL key                       # 剩余秒数
PTTL key                      # 剩余毫秒数
```

---

## 持久化

Redis 虽然是内存数据库，但提供两种持久化机制：

| 方式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **RDB** | 定时快照（`dump.rdb`） | 文件小、恢复快 | 可能丢最后一次快照后的数据 |
| **AOF** | 追加写命令日志 | 数据更安全，最多丢 1 秒 | 文件大、恢复慢 |

> 生产环境通常**两者都用**：RDB 做备份，AOF 保数据安全。

---

## 常见应用场景

### 1. 缓存（最常用）

```go
// 伪代码
val, err := redis.Get(ctx, "user:1001")
if err != nil {
    // cache miss → 查数据库
    user := db.GetUser(1001)
    redis.Set(ctx, "user:1001", user, 30*time.Minute)
    return user
}
return val
```

### 2. 分布式锁

```bash
SET lock:resource "uuid" NX EX 30
# NX = 不存在才设置（保证互斥）
# EX = 30秒自动过期（防止死锁）
```

### 3. 计数器 / 限流

```bash
INCR article:101:views      # 阅读量
# 滑动窗口限流用 ZSet 实现
```

---

## Redis vs Memcached

| 对比项 | Redis | Memcached |
|--------|-------|-----------|
| 数据类型 | 5 种 + Stream 等高级 | 仅 String |
| 持久化 | RDB/AOF | 纯内存 |
| 原子操作 | 事务 / Lua 脚本 | CAS |
| 集群 | Redis Cluster | 一致性哈希 |
| 内存效率 | 较高 | 更高（纯 KV） |

> 简单总结：**能选 Redis 就选 Redis**，功能全面且性能足够。

---

## 用 Go 连接 Redis

```go
package main

import (
    "context"
    "fmt"
    "github.com/redis/go-redis/v9"
)

func main() {
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    ctx := context.Background()

    rdb.Set(ctx, "greeting", "Hello Jinn!", 0)
    val, _ := rdb.Get(ctx, "greeting").Result()
    fmt.Println(val) // Hello Jinn!
}
```
