---
title: "慢 SQL 优化：从 EXPLAIN 看执行计划"
slug: "slow-sql-explain"
date: 2026-05-11
draft: false
description: "优化慢 SQL 的第一步不是改 SQL，而是看懂执行计划。本文逐列拆解 EXPLAIN 输出，并用一条真实慢查询演示从诊断到改写的完整过程。"
tags: ["SQL 优化", "EXPLAIN", "MySQL", "索引"]
categories: ["数据库"]
---

## 引言：别急着改 SQL

面对一条跑了几秒甚至几十秒的 SQL，很多人的第一反应是"加个索引试试"。这种拍脑袋的做法有时碰巧有效，更多时候是在碰运气。优化的正确姿势是先看执行计划——它告诉你优化器实际打算怎么执行：走哪个索引、扫描多少行、如何排序、怎样关联。读懂它，优化才有依据。

`EXPLAIN` 是 MySQL 提供的执行计划工具，它模拟优化器的决策过程并输出结果，不需要真正执行 SQL。下面先逐列讲清楚这些输出的含义，再用一个完整案例串起整个优化流程。

## EXPLAIN 各列含义

执行 `EXPLAIN` 会返回多行结果（每个表一行）。重点关注这几列：

- **`type`**：访问类型，是判断好坏的首要指标。从优到劣大致为 `system > const > eq_ref > ref > range > index > ALL`。`ALL` 代表全表扫描，通常意味着索引缺失或失效；至少应优化到 `range` 级别。
- **`key`**：实际选用的索引。为 `NULL` 说明没用上索引，`possible_keys` 则列出理论可用的索引。
- **`rows`**：预估需要扫描的行数。这是优化器基于统计信息的估算，越小越好。
- **`Extra`**：附加信息，包含很多关键线索。`Using filesort` 表示额外排序，`Using temporary` 表示用了临时表，`Using index` 表示覆盖索引（好事），`Using where` 表示做了条件过滤。

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10086;
```

## Join 驱动顺序与索引选择

多表关联时，`id` 表示 SELECT 的序号，**值越大越先执行**；`id` 相同才按从上到下的顺序执行。漏掉"越大越先"这个方向，很容易把 EXPLAIN 输出当成从上往下读的流程，把执行顺序理解反。

优化器通常倾向于把结果集小的表选作"驱动表"先执行，再用它的结果去关联被驱动表——注意这是**成本估算的结果，不是规则**，统计信息不准时它会算错。所以**被驱动表的关联字段上一定要有索引**，否则每次关联都要扫一遍。

一个常见问题是驱动表选择错误：当统计信息不准确时，优化器可能把大表当驱动表，导致灾难性的性能。可以用 `STRAIGHT_JOIN` 强制驱动顺序，但这是最后的手段，优先应修正统计信息。

## 统计信息对优化器的影响

MySQL 优化器是"基于成本"的，它依赖统计信息估算不同执行路径的代价。当表中数据大量增删后，统计信息可能失真，导致优化器做出错误选择——比如明明该走索引却选了全表扫描。这时执行：

```sql
ANALYZE TABLE orders;
```

更新统计信息，往往能让优化器重新选中正确的执行计划。这是很多"SQL 没改却突然变快"的幕后功臣。在 MySQL 8.0 中，还可以用 `EXPLAIN ANALYZE` 让优化器真正执行 SQL，输出各步骤的实际耗时与真实行数，用来对比估算值是否准确——这是排查"优化器误判"的利器。

## 覆盖索引与回表

InnoDB 的普通索引叶子节点存的是主键值。通过普通索引查到主键后，还要回主键索引再查一次完整行数据，这就是"回表"。如果要查询的字段都包含在索引里，就能直接返回结果、省去回表，这叫**覆盖索引**，在 EXPLAIN 的 `Extra` 里显示为 `Using index`。善用覆盖索引能显著减少 IO。

## 完整优化过程

假设有一张订单表，运营反馈"按用户查某状态订单并排序"很慢：

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  status TINYINT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  created_at DATETIME NOT NULL,
  KEY `idx_user` (`user_id`)
) ENGINE=InnoDB;

-- 原始慢 SQL
SELECT id, amount, created_at FROM orders
WHERE user_id = 10086 AND status = 2
ORDER BY created_at DESC;
```

**第一步，看执行计划：**

```sql
EXPLAIN SELECT id, amount, created_at FROM orders
WHERE user_id = 10086 AND status = 2
ORDER BY created_at DESC;
```

结果中 `key` 为 `idx_user`，`type` 为 `ref`，但 `Extra` 出现 `Using filesort`——说明用 `idx_user` 过滤后还要额外排序，且需要回表取 `amount`、`created_at`。当用户订单量大时，排序和回表开销都很高。

**第二步，设计覆盖索引。** 把查询涉及的字段纳入联合索引，并让排序列在末尾：

```sql
ALTER TABLE orders ADD KEY `idx_user_status_time` (`user_id`, `status`, `created_at`, `amount`);
```

**第三步，验证效果。** 再次 EXPLAIN，`Extra` 应变为 `Using index`（覆盖索引，不再回表，也没有 filesort），排序借助索引的有序性直接完成。`rows` 会下降，但降多少取决于 `status = 2` 的选择性——可能小几倍，也可能几乎没变。**真正的收益来自"不回表 + 不额外排序"，不是 `rows` 的数字好看**，所以别把 rows 当成唯一验收指标。

```sql
EXPLAIN SELECT id, amount, created_at FROM orders
WHERE user_id = 10086 AND status = 2
ORDER BY created_at DESC;
-- key = idx_user_status_time, Extra = Using index
```

## 踩坑与最佳实践

不要迷信 `rows` 的绝对值，它是估算值，关键是看优化前后和不同方案之间的相对变化。`Using filesort` 并非总是坏的——数据量小时内存排序很快，不能见到就恐慌。索引也非越多越好：本例新增索引后，应评估是否可删掉旧的 `idx_user`，避免写入负担和空间浪费。

另外，优化要在真实数据量上验证。开发环境的几百行数据看不出差异，优化器甚至可能因为表太小而选择全表扫描，导致误解执行计划。

## 小结

慢 SQL 优化有一条清晰路径：先用 `EXPLAIN` 看懂访问类型、扫描行数、Extra 线索，识别瓶颈在过滤、关联还是排序；再针对性设计索引——遵循最左前缀、把排序列和查询列纳入联合索引以实现覆盖、保持统计信息新鲜。把"看计划 → 找瓶颈 → 改索引 → 再验证"变成肌肉记忆，慢 SQL 优化就从玄学变成了工程。
