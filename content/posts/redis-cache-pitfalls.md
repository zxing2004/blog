---
title: "Redis 缓存三大问题：穿透、击穿、雪崩"
slug: "redis-cache-pitfalls"
date: 2026-04-09
draft: false
description: "缓存用得好是性能加速器，用不好是数据库杀手。本文拆解穿透、击穿、雪崩三大经典问题的成因，并给出布隆过滤、互斥锁、逻辑过期等可落地的解决方案。"
tags: ["Redis", "缓存", "布隆过滤器", "高并发"]
categories: ["数据库"]
---

## 引言：缓存挡在数据库前面

引入 Redis 的初衷，是让高频读请求在内存层就被拦截，避免打到脆弱的数据库。但在高并发或异常流量下，缓存层可能瞬间"失效"，让所有请求涌向数据库——轻则响应变慢，重则数据库直接被打挂。生产环境里，这类事故往往不是单点故障，而是某个边缘条件被放大后的连锁反应。

穿透、击穿、雪崩是三个最典型的场景。它们的共同表象是"缓存没挡住"，但成因和应对策略完全不同。把它们混为一谈，容易开出错误的药方。下面逐一拆解。

## 三种问题的定义与成因

**缓存穿透**：查询一个根本不存在的数据。缓存中没有，数据库也查不到，于是每次请求都穿透缓存直达数据库。典型触发场景是恶意构造大量不存在的 ID，或业务侧脏数据。它的特征是"查不到"，且请求分散在大量不同的键上。

**缓存击穿**：某个热点 Key 在过期的瞬间，恰好有大量并发请求涌入。这些请求同时发现缓存未命中，一起涌向数据库。与穿透不同，击穿针对的是"存在且热门"的单个 Key。它的特征是"单点、瞬时、高并发"。

**缓存雪崩**：大量 Key 在同一时间段集中过期，或 Redis 实例整体宕机，导致短时间内海量请求全部落到数据库。相比击穿的单点，雪崩是"大面积"的灾难。

## 缓存穿透：布隆过滤器与空值缓存

对付穿透有两道防线。第一道是布隆过滤器：在写入数据时把 Key 的哈希映射到位图中，查询前先用它判断 Key 是否存在，不存在就直接返回，避免访问数据库。第二道是空值缓存：即使数据库查不到，也往缓存写一个带短过期时间的空值占位。

```python
# 伪代码：布隆过滤器 + 空值缓存
from redisbloom.client import Client
import json

rb = Client(host='localhost', port=6379)

def get_user(uid: int):
    cache_key = f"user:{uid}"
    # 第一道防线：布隆过滤器判定"不存在"则直接返回
    if not rb.bfExists('user_filter', str(uid)):
        return None

    cached = redis.get(cache_key)
    if cached is not None:
        return None if cached == b'NULL' else json.loads(cached)

    user = db.query("SELECT * FROM user WHERE id=%s", uid)
    if user is None:
        # 第二道防线：空值缓存，短 TTL 防止长期占位
        redis.setex(cache_key, 60, 'NULL')
    else:
        redis.setex(cache_key, 3600, json.dumps(user))
    return user
```

对应的 Redis 命令（使用 RedisBloom 模块）如下：

```bash
# 创建布隆过滤器，容错率 0.01，预计元素 100 万
BF.RESERVE user_filter 0.01 1000000
# 写入元素
BF.ADD user_filter 10086
# 判断是否存在
BF.EXISTS user_filter 10086
```

## 缓存击穿：互斥锁与逻辑过期

击穿的核心是"同一个热点 Key 过期瞬间被并发击穿"。有两种主流方案：

**互斥锁（mutex key）**：只让一个线程去重建缓存，其他线程阻塞等待后直接从缓存读取。优点是数据强一致，缺点是等待期间吞吐下降。

```python
import time, uuid

def get_hot_product(pid: int):
    key = f"product:{pid}"
    data = redis.get(key)
    if data is not None:
        return json.loads(data)

    # 抢重建锁，SETNX + 短过期防死锁
    lock_key = f"lock:{pid}"
    token = uuid.uuid4().hex
    if redis.set(lock_key, token, nx=True, ex=10):
        try:
            data = db.query("SELECT * FROM product WHERE id=%s", pid)
            redis.setex(key, 3600, json.dumps(data))
            return data
        finally:
            # 用 Lua 校验 token 后删除，避免误删他人的锁
            redis.eval("if redis.call('get',KEYS[1])==ARGV[1] "
                       "then return redis.call('del',KEYS[1]) else return 0 end",
                       1, lock_key, token)
    else:
        # 没抢到锁，短暂等待后重试
        time.sleep(0.05)
        return get_hot_product(pid)
```

**逻辑过期**：缓存值里携带一个过期时间字段，物理上永不过期。发现逻辑过期时返回旧值，同时后台异步重建。优点是请求不阻塞，缺点是有短暂的数据不一致窗口。

两种方案如何选择，取决于业务对一致性的要求：互斥锁适合库存、账户这类强一致场景，代价是等待期间的吞吐下降；逻辑过期适合新闻、商品详情这类允许短暂陈旧的场景，代价是不一致窗口。还有一个容易被忽视的事实：缓存层的高可用并不等于数据不丢。Redis 主从切换、网络分区期间，缓存可能出现短暂不可用或数据回退，同样会有请求涌向数据库。因此三大问题的防护手段应与限流、降级配合形成完整防线，而不能指望单靠缓存技巧解决所有问题。

## 缓存雪崩：过期时间随机化与多级缓存

雪崩源于"同时失效"。最直接的对策是给过期时间加随机抖动，让 Key 的失效时间分散开：

```bash
# 基础 TTL 3600 秒，叠加 0~300 秒随机值
SET product:1001 "{...}" EX 3753
SET product:1002 "{...}" EX 3897
```

```python
import random
ttl = 3600 + random.randint(0, 300)
redis.setex(key, ttl, json.dumps(data))
```

更稳妥的方案是搭建多级缓存（本地缓存 Caffeine + Redis），Redis 挂掉时本地缓存还能顶一阵；同时对 Redis 做高可用（哨兵或 Cluster），并在应用层做限流降级，为数据库争取缓冲时间。

## 踩坑与最佳实践

布隆过滤器不是银弹：它存在误判率（可能把不存在的判为存在），且不支持删除（可用计数布隆过滤器变通）。空值缓存要设置较短 TTL，否则大量空键会浪费内存。互斥锁的锁过期时间必须大于重建耗时，否则会出现并发重建；释放锁务必用 Lua 校验 token，杜绝误删。逻辑过期适合对一致性要求不高的热点数据，例如商品详情，但不适合库存、价格这类敏感场景。

## 小结

三种问题的关键词各不相同：穿透是"查不存在的键"，用布隆过滤器和空值缓存拦截；击穿是"热点键过期瞬间"，用互斥锁或逻辑过期化解；雪崩是"大面积同时失效"，用随机化过期时间、多级缓存和高可用架构兜底。理解成因后按需组合，才能既保护数据库，又不牺牲缓存带来的性能红利。
