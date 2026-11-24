---
title: "PostgreSQL 与 MySQL 该怎么选"
slug: "postgresql-vs-mysql"
date: 2026-04-23
draft: false
description: "PostgreSQL 和 MySQL 都是顶级开源关系库，但设计哲学截然不同。本文从特性、生态到场景逐项对比，给出一套可落地的选型决策框架。"
tags: ["PostgreSQL", "MySQL", "数据库选型", "架构"]
categories: ["数据库"]
---

## 引言：没有最好，只有最合适

"PostgreSQL 和 MySQL 选哪个"是技术选型会上经久不衰的争论。争论往往演变成信仰之争——一方强调 PG 的功能强大与标准合规，另一方强调 MySQL 的生态成熟与简单高效。但脱离业务场景谈优劣没有意义：两者都是久经考验的生产级数据库，选择的关键在于哪一个更契合你的数据模型、查询模式和团队能力。

选型失误的代价往往在项目中后期才显现：错误的存储引擎选型、缺失的扩展能力、团队不熟悉的调优手段，都会变成技术债。下面从设计哲学出发，逐项对比两者的能力边界，最后落到具体的决策建议。

## 定位差异：学院派 vs 实用派

MySQL 最初为 Web 应用而生，设计目标是简单、快速、易运维。它采用可插拔存储引擎架构（InnoDB、MyISAM 等），把"存储"与"服务层"解耦，这种务实取向让它成为 LAMP 栈的事实标准，互联网公司积累了海量运维经验。

PostgreSQL 则出身学院派，自称"世界上最先进的开源关系数据库"。它严格遵循 SQL 标准，提供丰富的数据类型、强大的扩展机制和 MVCC 实现。PG 更像是"可以存一切"的数据平台，而不仅是 SQL 执行引擎。

一句话概括：MySQL 追求"够用且高效"，PostgreSQL 追求"功能完备且正确"。

## 关键特性对比

**JSON 支持**：这里有个流传很广的误解——"PG 的 JSONB 是二进制存储，MySQL 的 JSON 是文本所以慢"。不成立：MySQL 自 5.7 起就把 JSON 存成一种内部二进制格式，读取单个元素不需要解析整段文本。真正的差距不在存储格式，而在**索引能表达什么查询**。

PG 的 `JSONB` 可以建 GIN 索引，直接加速包含（`@>`）、键存在（`?`）、键集合（`?|`、`?&`）这类**结构性判断**：

```sql
-- PostgreSQL：JSONB 字段 + GIN 索引
CREATE TABLE events (
  id BIGSERIAL PRIMARY KEY,
  payload JSONB NOT NULL
);
-- 默认 jsonb_ops 支持 @> ? ?| ?&；只做 @> 时可用 jsonb_path_ops，索引更小更快
CREATE INDEX idx_payload ON events USING GIN (payload);

-- 查询 payload 中 metadata.level = 'error' 的记录
SELECT * FROM events
WHERE payload @> '{"metadata": {"level": "error"}}';
```

注意 GIN 加速的是上面这种"整棵子树是否包含"的判断。如果要按抽出来的标量值做等值比较，GIN 帮不上忙，得建表达式索引（btree）：

```sql
CREATE INDEX idx_err_level ON events ((payload #>> '{metadata,level}'));
SELECT * FROM events WHERE payload #>> '{metadata,level}' = 'error';
```

MySQL 这一侧，同样的需求在 8.0.13 之后有两条路。第一条是**函数索引**，不再需要先加一张生成列：

```sql
-- MySQL 8.0.13+：直接对表达式建索引
-- 注意 -> > 返回 LONGTEXT 不能直接索引，必须 CAST 成定长类型
ALTER TABLE events
  ADD INDEX idx_err_level ((CAST(payload->>'$.metadata.level' AS CHAR(20))));

-- 查询里的表达式要跟索引表达式逐字对上，优化器才会用
SELECT * FROM events
WHERE CAST(payload->>'$.metadata.level' AS CHAR(20)) = 'error';
```

第二条是 8.0.17 引入的**多值索引**，专门解决"JSON 数组里任一元素命中"的场景——这在 PG 里要写 `@>` 或展开成 `jsonb_array_elements` 再 join，反而不如索引直接：

```sql
-- MySQL 8.0.17+：给 JSON 数组建多值索引
ALTER TABLE events
  ADD INDEX idx_tags ((CAST(payload->'$.tags' AS CHAR(20) ARRAY)));

SELECT * FROM events WHERE 'mysql' MEMBER OF (payload->'$.tags');
```

所以两边的差异是**形状不同，不是高下**：PG 用 GIN 覆盖结构性子树匹配，运算符和 jsonpath 表达力更强，一条 `@>` 能描述很复杂的嵌套条件；MySQL 的索引必须绑定到某个具体路径/表达式上，复杂嵌套要么拆多个索引，要么在应用层拼 SQL。前者在"查询形态不确定、由用户拼条件"的场景更省事，后者在"路径固定、访问模式清楚"的场景反而索引更精准、写入代价更小（GIN 的插入放大是实打实的）。

**扩展生态**：PG 拥有 PostGIS（地理信息）、pg_trgm（模糊匹配）、TimescaleDB（时序）等大量扩展，可以在数据库内完成原本需要额外组件的工作。MySQL 的插件生态相对单薄。

**事务与锁**：两者都支持 ACID，普通 `SELECT` 都不加锁、不阻塞写入，所以"读不阻塞写"不是 PG 的独有能力。真正的差别在**旧版本存在哪**：

- PG 把旧版本留在堆表里，靠 `VACUUM` 回收。好处是回滚几乎瞬时（标记即可）、`BEGIN; DROP TABLE ...; ROLLBACK;` 这类事务性 DDL 成立；代价是更新频繁的表会膨胀，长事务会拖住 xmin 使死元组清不掉，索引也跟着膨胀。
- InnoDB 把旧版本放在 undo log 里，由 purge 线程回收。好处是表本身不因更新而膨胀；代价是长事务会撑大 history list，回滚大事务要等 undo 回放，DDL 也无法回滚（8.0 的原子 DDL 解决的是"崩溃后不留半截"，不是"能撤销"）。

锁行为上还有一处常被忽略的差异：InnoDB 在 `REPEATABLE READ` 下对范围条件加 next-key 锁（记录锁 + 间隙锁），并发写同一区间容易死锁；PG 没有间隙锁，靠"更新冲突时后到者报错重试"来保证可串行化，不会因为间隙互相卡住，但在高并发点更新同一行时更容易出现序列化失败。两种模型的失败模式不同，压测时看到的报错文本也不同。

**复制与高可用**：MySQL 的 binlog 复制生态极其成熟，主从、双主、MGR、各类中间件方案丰富，运维工具链完善。PG 的流复制同样可靠，配合 Patroni、pgBouncer 等工具可以构建高可用集群，但生态复杂度较高。

**全文检索与 GIS**：PG 内置 `tsvector` 全文检索，配合 `pg_jieba`、`zhparser` 等扩展可做中文分词；PostGIS 是 GIS 领域的事实标准，函数覆盖面其他开源库难比。MySQL 这一侧并非没有：InnoDB 全文索引自 5.6 起就有，5.7 加的 ngram 解析器能处理中日韩（不需要按空格分词），8.0 的空间索引支持 SRID 与 `ST_Contains` 等常用判断，做"附近的门店"这类需求够用。差距在于复杂几何运算与坐标系转换——那部分 PostGIS 是另一个量级。

下表按 2026 年初的两个长期支持版本横向对照（MySQL 8.4 LTS / PostgreSQL 17）。小版本行为差异很大，具体特性以官方文档为准。

| 维度 | MySQL 8.4 LTS | PostgreSQL 17 |
| --- | --- | --- |
| 数据模型 | 关系型为主；JSON 走二进制格式，索引需绑定具体路径 | 关系 + JSONB + 数组 + 范围/复合/自定义类型 |
| JSON 索引 | 函数索引（8.0.13+）、多值索引（8.0.17+），需绑定路径 | GIN 支持整棵子树包含判断，另有表达式索引 |
| 扩展能力 | 有限，多靠外部组件补齐 | 极强（PostGIS、TimescaleDB 等） |
| 全文检索 | InnoDB 全文索引 + ngram 中文分词 | 内置 tsvector + 第三方中文分词扩展 |
| 旧版本存储 | undo log，表不膨胀但 history list 会涨 | 堆表内保留，需 VACUUM，长事务致膨胀 |
| DDL | 原子 DDL（崩溃安全），不可回滚 | 事务性 DDL，可随事务回滚 |
| 复制生态 | 极成熟，工具丰富 | 成熟，但更复杂 |
| 运维门槛 | 较低，普及度高 | 较高，专业人才相对少 |
| 典型场景 | Web OLTP、互联网业务 | 复杂查询、GIS、分析混合 |

## 场景取舍

**OLTP 为主、简单读写**：如电商交易、用户中心，MySQL 是稳妥选择。它的生态、中间件、云服务和运维经验最丰富，遇到问题容易找到解决方案。

**复杂查询、数据分析与混合负载**：PG 优势明显。强大的优化器、丰富的数据类型与窗口函数，让它在报表、analytics、复杂统计场景中更省力。

**地理信息、时序、全文检索**：优先考虑 PG 及对应扩展，往往一个数据库就能搞定整套需求。

**团队因素不可忽视**：如果团队长期用 MySQL，贸然切换到 PG 会带来陡峭的学习成本和运维风险。技术先进性必须让位于团队的掌控能力。

## 踩坑与最佳实践

一个常见误区是"PG 性能不如 MySQL"。事实上在多数 OLTP 场景两者差距不大，性能更多取决于 schema 设计、索引和 SQL 质量。跑分数据必须在你的真实负载下验证，不能照搬网络结论。

另一个坑是盲目追求"功能全"。PG 的扩展虽强，每引入一个都增加运维复杂度。另一个现实问题是人才市场：MySQL 工程师供给更充足，PG 高手相对稀缺，这会影响招聘与成本。

## 小结

选型可以归结为三问：**数据模型是否复杂**（大量 JSON、GIS、数组 → 倾向 PG）、**查询是否重型**（复杂分析、全文检索 → 倾向 PG）、**团队能否驾驭**（MySQL 生态成熟、上手快 → 倾向 MySQL）。没有放之四海皆准的答案，把业务特征与团队现实摆在桌面上，答案通常就清晰了。若仍犹豫，用真实业务负载做一次 POC 压测，比任何争论都更有说服力。
