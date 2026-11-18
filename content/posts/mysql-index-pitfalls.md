---
title: "MySQL 索引失效的七种典型场景"
slug: "mysql-index-pitfalls"
date: 2026-03-12
draft: false
description: "明明建了索引，查询却依然全表扫描。本文从 B+ 树结构出发，拆解七种最常见的索引失效场景，并给出可运行的反例与改写方案。"
tags: ["MySQL", "索引", "EXPLAIN", "SQL 优化"]
categories: ["数据库"]
---

## 引言：索引建了，为什么还是慢

在慢 SQL 排查中，最让人困惑的往往不是"没建索引"，而是"索引明明在，优化器却不用"。一条几千万行的表查询，开发人员信誓旦旦地说字段上有索引，结果 `EXPLAIN` 里 `type` 赫然写着 `ALL`，`rows` 扫了整张表。这种落差几乎都指向同一类根因：SQL 的写法让索引变得"不可用"。

索引不是建上就一劳永逸的。它是优化器在特定条件下才会选择的一条访问路径，任何一个破坏条件的写法，都会让这条路被放弃。理解失效的成因，比记住"哪些写法不能用"更重要。下面先从索引的底层结构讲起，再逐一拆解七种典型场景。

## B+ 树：理解失效的钥匙

InnoDB 的默认索引是 B+ 树结构。数据按索引键值有序地存放在叶子节点，非叶子节点只存键值和指针。对联合索引 `(a, b, c)` 而言，InnoDB 先按 `a` 排序，`a` 相同时按 `b` 排序，`b` 相同时再按 `c` 排序。这个"有序"是分层的：只有当 `a` 的值确定后，`b` 的有序性才有意义。

这个结构直接决定了索引能否被使用。索引查找本质是一次"在有序序列里定位"的过程——你必须给出从左边界开始的、连续且确定的键值。一旦查询条件破坏了"从最左列开始"或"键值确定"这两个前提，B+ 树的有序性就无法被利用，优化器只能退回到全表扫描。七种失效场景，本质上都是这两个前提被破坏的变体。

还需要补充一点：B+ 树的叶子节点通过双向链表相连，因此范围查询（BETWEEN、大于、小于）和 ORDER BY 都能借助索引的顺序性完成。这也解释了"范围查询之后的列失效"——一旦某一列用范围定位，它之后的列就不再是连续确定的区间，B+ 树的定位能力到此为止，后续条件只能逐行过滤。MySQL 8.0 可借助索引条件下推（ICP）在引擎层提前过滤、减少回表次数，但无法恢复有序定位。

我们先准备一张示范表，后面的例子都基于它：

```sql
CREATE TABLE `orders` (
  `id` BIGINT NOT NULL AUTO_INCREMENT,
  `user_id` BIGINT NOT NULL,
  `order_no` VARCHAR(32) NOT NULL,
  `status` TINYINT NOT NULL,
  `amount` DECIMAL(10,2) NOT NULL,
  `created_at` DATETIME NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_union` (`user_id`, `status`, `created_at`),
  KEY `idx_order_no` (`order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 七种典型失效场景

**1. 违反最左前缀原则。** 联合索引 `idx_union` 从 `user_id` 起头，跳过它直接用 `status` 查询，索引失效：

```sql
-- 反例：跳过最左列
SELECT * FROM orders WHERE status = 1;
-- 改写：补上最左列，或为 status 单独建索引
SELECT * FROM orders WHERE user_id = 10086 AND status = 1;
```

**2. 隐式类型转换。** `order_no` 是字符串，传入数字会让 MySQL 对列做隐式转换，等同于在列上加函数：

```sql
-- 反例：字符串列用数字比较
SELECT * FROM orders WHERE order_no = 20260312001;
-- 改写：保持一致的字符串类型
SELECT * FROM orders WHERE order_no = '20260312001';
```

**3. 索引列使用函数或运算。** 对 `created_at` 套函数，或在 `amount` 上做运算，都会让索引失效：

```sql
-- 反例：列上套函数
SELECT * FROM orders WHERE DATE(created_at) = '2026-03-12';
-- 改写：把运算移到常量一侧，转成范围查询
SELECT * FROM orders
WHERE created_at >= '2026-03-12 00:00:00'
  AND created_at <  '2026-03-13 00:00:00';
```

**4. LIKE 以通配符开头。** 前缀 `%` 破坏了有序定位，后缀 `%` 则不影响：

```sql
-- 反例：前导通配符
SELECT * FROM orders WHERE order_no LIKE '%20260312';
-- 改写：使用后缀匹配
SELECT * FROM orders WHERE order_no LIKE '20260312%';
```

**5. OR 连接了非索引列。** OR 要求两侧都能走索引，只要有一列没索引，整个条件就退化：

```sql
-- 反例：amount 无索引，导致 order_no 的索引也用不上
SELECT * FROM orders WHERE order_no = 'A001' OR amount > 100;
-- 改写：拆成 UNION ALL，各自走索引
SELECT * FROM orders WHERE order_no = 'A001'
UNION ALL
SELECT * FROM orders WHERE amount > 100;
```

**6. 范围查询后的列失效。** 联合索引中，`created_at` 之后若还有列，范围查询会让后续列失效。这里 `idx_union` 的第三列是 `created_at`：

```sql
-- status 用了范围，其后的 created_at 无法用于定位
SELECT * FROM orders
WHERE user_id = 10086 AND status > 1 AND created_at = '2026-03-12 10:00:00';
-- 改写：把等值列尽量前置，或调整索引列顺序
ALTER TABLE orders ADD KEY `idx_uid_time` (`user_id`, `created_at`, `status`);
```

**7. 字符集或排序规则不一致。** 两表 JOIN 时关联字段的字符集/排序规则不同，会导致隐式转换。建表时需统一：

```sql
-- 检查两张表的字符集与排序规则
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'db_name' AND COLUMN_NAME = 'order_no';
-- 统一为 utf8mb4 / utf8mb4_0900_ai_ci（MySQL 8.0）
ALTER TABLE orders MODIFY order_no VARCHAR(32)
  CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

## 用 EXPLAIN 验证判断

所有猜测都必须用 `EXPLAIN` 落地。关注三个关键列：`type` 从好到坏大致是 `system > const > ref > range > index > ALL`，出现 `ALL` 就是全表扫描；`key` 显示实际选中的索引，`NULL` 代表没用索引；`Extra` 里的 `Using filesort`、`Using temporary` 往往意味着额外开销。

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id = 10086 AND status = 1;
```

对比改写前后的 `type` 与 `rows`，是判断优化是否生效最直接的方式。注意优化器基于成本做选择，即使索引可用，当命中行数占比很高时，它也可能主动放弃索引——这不是失效，而是合理的选择。

## 踩坑与最佳实践

一个高频误区是把函数索引、表达式索引当成万能药。MySQL 8.0 支持函数索引，但它有额外的维护成本，能改写 SQL 就优先改写。另一个坑是"索引越多越好"：每个索引都会拖慢写入、占用空间，联合索引要考虑列顺序与复用性，避免重复建设。

还有一点常被忽略：统计信息过期会误导优化器。当数据分布发生大变化后，执行 `ANALYZE TABLE orders;` 更新统计信息，往往能让优化器重新选中正确的索引。

## 小结

索引失效的七种场景，归根结底是破坏了 B+ 树"从最左列开始、键值确定"的定位前提。写法上守住几个原则即可：联合索引不跳最左列、保持列类型与参数类型一致、不在列上套函数或做运算、避免前导 `%`、谨慎使用 OR 与范围条件、统一字符集。最后，永远用 `EXPLAIN` 说话——它才是优化器真实意图的唯一真相来源。
