---
title: "Redis 持久化：RDB 与 AOF 的差异"
slug: "redis-rdb-vs-aof"
date: 2026-06-09
draft: false
description: "Redis 是内存数据库，断电后数据如何保全？本文讲透 RDB 快照与 AOF 日志的原理、触发方式和取舍，并给出混合持久化配置与验证命令。"
tags: ["Redis", "持久化", "RDB", "AOF"]
categories: ["数据库"]
---

## 引言：内存数据库的持久化难题

Redis 把所有数据放在内存里，读写极快，但进程一旦退出，内存数据就会丢失。为了让数据在重启后还能恢复，Redis 提供了两种持久化机制：RDB 和 AOF。它们代表了两种不同的思路——RDB 是"在某个时刻拍一张全量快照"，AOF 是"把每一条写命令都记录下来"。

很多开发者只是模糊地知道"要开持久化"，却不清楚两者的取舍，配置要么全用默认，要么听说 AOF 更安全就盲目开启。理解它们的差异，才能在数据安全、恢复速度和性能开销之间做出合理权衡。

## RDB：快照原理与触发

RDB（Redis Database）持久化会在某个时间点把内存中的数据集生成一个紧凑的二进制快照文件 `dump.rdb`。它的触发分两类：

- **手动触发**：`SAVE` 会在主线程执行，阻塞所有请求，生产环境基本不用；`BGSAVE` 则 fork 一个子进程在后台生成快照，主进程继续服务。
- **自动触发**：通过 `redis.conf` 中的 `save m n` 规则，例如 `save 900 1` 表示 900 秒内至少 1 次修改就触发一次 BGSAVE。

RDB 的优点是文件紧凑、恢复速度快（直接把二进制加载进内存），非常适合做冷备和灾难恢复。缺点也很明显：它是按周期快照，两次快照之间宕机会丢失这段时间的数据；而且每次 BGSAVE 都要 fork 子进程，数据量大时有一定开销。

## AOF：追加日志与 rewrite

AOF（Append Only File）会把每一条写命令以 Redis 协议格式追加到日志文件末尾。重启时，Redis 重新执行一遍 AOF 里的命令即可重建数据。由于记录的是命令序列，AOF 的数据安全性远高于 RDB。

AOF 的写入粒度由 `appendfsync` 控制：`always`（每条命令都 fsync，最安全但最慢）、`everysec`（每秒一次，折中方案，默认推荐）、`no`（由系统决定，最快但风险高）。

问题是 AOF 文件会不断膨胀——同一个 key 被改一百次，就有一百条记录。Redis 提供 `BGREWRITEAOF` 命令做 rewrite：fork 子进程，根据当前内存数据重新生成一份精简的 AOF（只保留能重建当前数据的最小命令集），然后替换旧文件。触发规则由 `auto-aof-rewrite-percentage` 和 `auto-aof-rewrite-min-size` 控制。

## 混合持久化

Redis 4.0 引入了混合持久化，兼顾 RDB 的快速恢复和 AOF 的低数据丢失。开启后，AOF 文件由两部分组成：文件开头是 RDB 格式的全量快照，之后是增量的 AOF 命令。rewrite 时，子进程先把当前数据以 RDB 格式写入，再把 rewrite 期间的增量命令以 AOF 格式追加。这样重启时先快速加载 RDB，再重放少量增量命令，恢复速度和安全性兼得。混合持久化由配置项 `aof-use-rdb-preamble yes` 开启（Redis 4.0 以上）。需要注意，混合模式下 AOF 文件前半段是二进制 RDB、后半段是文本命令，直接查看或人工编辑文件时要留意这种混合格式。

## fork 与 COW 的内存开销

RDB 的 BGSAVE 和 AOF 的 rewrite 都需要 fork 子进程。fork 采用写时复制（Copy-On-Write）：子进程和父进程共享同一份内存页，只有当父进程要修改某个页时，才复制出一份新的。这意味着 fork 瞬间几乎不额外占用内存，但随着父进程在快照期间不断写入，被修改的页会陆续复制，内存占用逐渐上升。

经验值是为 Redis 预留足够内存：在 BGSAVE/rewrite 期间，最坏情况下内存可能接近翻倍。如果机器内存紧张、写入又频繁，COW 可能触发 OOM。这也是为什么要在业务低峰期安排持久化、并监控内存水位的根本原因。

## redis.conf 关键配置与验证命令

```bash
# ---------- RDB 配置 ----------
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /var/lib/redis
stop-writes-on-bgsave-error yes   # 快照失败时停止写入，及时暴露问题

# ---------- AOF 配置 ----------
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes          # 开启混合持久化
```

修改配置后重启，用以下命令验证持久化状态：

```bash
# 查看持久化相关信息（rdb_last_save_time、aof_enabled、aof_rewrite_in_progress 等）
redis-cli INFO persistence

# 手动触发快照与 rewrite
redis-cli BGSAVE
redis-cli BGREWRITEAOF

# 查看最近一次持久化状态
redis-cli LASTSAVE

# 检查 AOF 文件
ls -lh /var/lib/redis/appendonly.aof
```

## 踩坑与最佳实践

一个典型误区是"开了 RDB 就高枕无忧"。RDB 按周期快照，宕机时可能丢失几分钟数据，对数据敏感的场景必须搭配 AOF 或使用混合持久化。另一个坑是把 `appendfsync` 设为 `always` 却抱怨性能差——每条命令都 fsync 会严重拖慢写入，除非对数据零丢失有硬性要求，否则 `everysec` 是更务实的选择。

还要警惕 `stop-writes-on-bgsave-error`：设为 `yes` 时，一旦磁盘满导致快照失败，Redis 会拒绝写入。这看似激进，实则避免"数据没落盘却仍在写入"的假安全，应保持开启并配置磁盘告警。

## 小结

RDB 是紧凑的全量快照，恢复快但可能丢失数据，适合冷备；AOF 记录每条写命令，数据安全但文件大、恢复慢，rewrite 用来给它瘦身。混合持久化把两者结合，是当前生产环境的主流推荐。无论选择哪种，都要为 fork/COW 预留内存、监控持久化失败告警，并定期做恢复演练——持久化配置的意义，只有在成功恢复的那一刻才真正兑现。
