---
title: "事务隔离级别与 MVCC 到底怎么工作"
slug: "mysql-mvcc-isolation"
date: 2026-03-26
draft: false
description: "脏读、不可重复读、幻读到底有什么区别？MVCC 又是如何让读写并发互不阻塞的？本文用可复现的事务序列讲透 InnoDB 的隔离实现。"
tags: ["MySQL", "事务", "MVCC", "锁"]
categories: ["数据库"]
---

## 引言：并发下的"看到什么"

两个事务同时操作同一批数据，它们彼此能看到什么，是数据库最核心也最容易混淆的问题。业务上常见的困惑是：明明在自己事务里查了一次，另一个事务提交后，再查怎么就变了？或者两个事务"同时"插入，为什么有的会报死锁？

这些现象背后是隔离级别与 MVCC 的共同作用。隔离级别定义了"应该看到什么"的规则，MVCC（多版本并发控制）则是 InnoDB 实现这套规则的机制。只背四个隔离级别的名字没意义，理解 MVCC 如何在不加锁的情况下提供一致性读，才能真正解释生产环境中的诡异现象。

## ACID 与四种隔离级别

事务的 ACID 中，隔离性（Isolation）追求的是：并发事务之间尽量互不干扰。但完全串行执行性能太差，SQL 标准因此定义了四种隔离级别，允许不同程度的"干扰"：

- **读未提交（READ UNCOMMITTED）**：能读到其他事务未提交的数据。
- **读已提交（READ COMMITTED）**：只能读到已提交的数据。
- **可重复读（REPEATABLE READ）**：同一事务内多次读取同一行结果一致，这是 InnoDB 的默认级别。
- **串行化（SERIALIZABLE）**：所有读操作都加锁，强制串行。

隔离级别越高，并发异常越少，但并发性能也越低。InnoDB 的高明之处在于，它在"可重复读"级别下，借助 MVCC 和间隙锁，用接近读已提交的并发成本，达到了比标准更强的隔离效果。

## 三种并发异常

理解隔离级别，先要理解它要防的是什么：

- **脏读**：读到其他事务尚未提交、随后又回滚的数据。
- **不可重复读**：同一事务内两次读取同一行，因中间有其他事务提交更新，结果不一致——重点在"更新"。
- **幻读**：同一事务内两次范围查询，因中间有其他事务插入新行，后者读到了"幻影行"——重点在"插入或删除"。

## MVCC 的实现：隐藏列、undo log 与 ReadView

InnoDB 在每行记录后追加两个隐藏列：`DB_TRX_ID`（最后修改该行的事务 ID）和 `DB_ROLL_PTR`（回滚指针，指向 undo log 中的旧版本）。当一行被多次修改，这些 undo log 会通过回滚指针串成一条版本链，链头是最新版本。

事务发起一次"快照读"（普通 SELECT）时，InnoDB 会生成一个 **ReadView**，它记录当前活跃（未提交）事务的 ID 集合，以及 `min_trx_id`、`max_trx_id`。判断某个版本是否对当前事务可见，遵循一套规则：如果版本的 `DB_TRX_ID` 小于 `min_trx_id`，说明修改者早已提交，可见；如果大于等于 `max_trx_id`，说明是快照之后才开启的事务修改的，不可见；如果落在活跃集合里，说明修改时还未提交，不可见，需沿版本链继续往回找。

关键差异在于 ReadView 的生成时机：**读已提交**每次快照读都生成新 ReadView，所以能看到其他事务新提交的数据（出现不可重复读）；**可重复读**只在事务第一次快照读时生成一次，之后复用同一个，因此整个事务看到的是同一份快照。

## 快照读与当前读

并非所有读都走 MVCC。普通的 `SELECT ... WHERE` 是快照读，读的是历史版本、不加锁。而 `SELECT ... FOR UPDATE`、`LOCK IN SHARE MODE`，以及 `UPDATE`、`DELETE`、`INSERT` 都是当前读，它们读取最新版本并加锁。理解这一点，才能解释"为什么快照读看不到，当前读却能看到"的现象：

```sql
-- 示范表
CREATE TABLE account (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(32),
  balance INT
) ENGINE=InnoDB;
INSERT INTO account(name, balance) VALUES ('Alice', 1000), ('Bob', 1000);

-- 事务 A（可重复读）：快照读
START TRANSACTION;
SELECT balance FROM account WHERE name = 'Alice';  -- 1000，生成 ReadView

-- 此时事务 B 执行并提交：
-- START TRANSACTION;
-- UPDATE account SET balance = 800 WHERE name = 'Alice';
-- COMMIT;

-- 事务 A 再次快照读：仍是 1000（不可重复读被避免）
SELECT balance FROM account WHERE name = 'Alice';

-- 但当前读会看到最新值 800
SELECT balance FROM account WHERE name = 'Alice' FOR UPDATE;

COMMIT;
```

## 间隙锁与幻读

在可重复读级别，快照读由 MVCC 保证不出现幻读（读到的是快照）。但当前读必须加锁才能防止幻读，这靠的是 **间隙锁（Gap Lock）** 与行锁的组合——合称 Next-Key Lock。它不仅锁住已有的行，还锁住行与行之间的间隙，阻止其他事务在间隙中插入新行：

```sql
-- 事务 A：范围当前读，锁定 (10, 20] 及其间隙
START TRANSACTION;
SELECT * FROM account WHERE id BETWEEN 10 AND 20 FOR UPDATE;

-- 事务 B 此时插入 id=15 会被阻塞，直到 A 提交
-- INSERT INTO account(id, name, balance) VALUES (15, 'Carol', 500);
COMMIT;
```

## 踩坑与最佳实践

一个常见误区是认为"可重复读就完全没有并发问题"。实际上，快照读与当前读混用可能导致数据不一致——典型场景是"先读后写"：快照读判断余额足够，再用当前读更新，中间数据可能已被改动。正确做法是用 `SELECT ... FOR UPDATE` 做判断，让读也变成当前读并加锁。

另一个坑是间隙锁在并发插入时容易产生死锁，尤其是唯一键冲突场景。保持事务短小、按固定顺序访问资源、避免大范围加锁，能显著降低死锁概率。

## 小结

四种隔离级别定义了并发可见性的规则，MVCC 通过隐藏列、undo log 版本链和 ReadView，让快照读在大多数场景下无需加锁即可提供一致性视图。可重复读在首次快照读时固定 ReadView，从而避免不可重复读；当前读则依赖 Next-Key Lock（行锁加间隙锁）阻止幻读。理解"快照读看版本链、当前读看最新并加锁"这条主线，生产环境中的并发疑案就能逐一破解。
