---
title: "Agent 长任务的可靠性：检查点、幂等与断点续跑"
date: 2026-09-28
draft: false
description: "长程异步 Agent 的失败是常态而非例外。本文讲透检查点、幂等与断点续跑三大机制，并给出可直接落地的 Python 实现与工程避坑清单。"
tags: ["Agent", "可靠性", "幂等", "断点续跑"]
categories: ["后端与架构"]
---

## 热点引入：当 Agent 开始「上长班」

2026 年 9 月，Cognition 发布了面向长程异步软件工程的新模型（官方定位是"把需求丢进去，几小时后交付一个可用的分支"，具体命名与指标以官方公告为准）。这类任务的共同特征是：步骤数以百计、跨越数十次工具调用、运行期间模型与外部世界持续交互。能力叙事之外，一个更朴素的问题浮出水面：长任务几乎必然中途失败。

上下文窗口会溢出，某个工具调用会超时，进程会被 OOM killer 杀掉，网络会抖。短任务失败重跑即可，长任务重来的代价可能是几小时的计算和一笔不小的账单。可靠性，从「加分项」变成了「能不能用」的前提。

本文聚焦三个相互咬合的机制：检查点（checkpoint）、幂等（idempotency）与断点续跑（resume）。它们分别回答三个问题：状态怎么存、失败后怎么安全地重来、以及怎么接着干而不是从头来。这是把 Agent 从 Demo 推向生产系统的分水岭。

## 技术原理：长程 Agent 为什么会「掉链子」

先给失败分类，否则优化无从下手。长程 Agent 的失败大致四类：

- **上下文溢出**：历史消息无限增长，终将撞上 token 上限；即便不撞，关键信息也会被海量中间输出稀释，导致「迷失在中间」。
- **步骤失败**：单次工具调用可能超时、被限流、返回脏数据，这是分布式系统的常态，不是意外。
- **重复执行**：崩溃恢复后若简单重试，已提交的副作用（发帖、转账、改文件）会被执行两次，酿成事故。
- **状态丢失**：进程一挂，内存里的对话历史、计划、中间结果全部蒸发，几小时的工作归零。

对应三道防线。

**检查点**是把每一步的「输入—决策—工具结果—状态快照」持久化。它不是简单写日志，而是可重建执行现场的最小完备状态：给定检查点，系统应能恢复到一个一致的、可继续推进的状态。实践中常把状态机当前状态、已完成任务、压缩后的上下文摘要、工具结果缓存一起落库。

**幂等性**是让工具「重试安全」。核心手段是给每个操作分配唯一 `operation_id`，由服务端或本地去重层做仲裁：同一 ID 的重复请求直接返回首次结果，不再产生副作用。没有幂等，断点续跑就是一个危险动作——你无法确定崩溃发生在副作用提交之前还是之后。

**断点续跑**是启动时从最近一个有效检查点恢复，而非重头执行。它把「重跑成本」从 O(任务总长) 降到 O(剩余步数)。三者的关系可以概括为一句话：检查点提供「从哪续」，幂等保证「续得安全」，断点续跑是前两者叠加后的收益。

此外还有两个保护阀：**最大步数**防止 Agent 陷入自我重复的死循环，**单步超时**防止一次调用无限挂起。它们与状态机配合——每一步都是状态机的一次合法迁移，非法迁移直接拒绝并落错，避免带着脏状态继续跑。

## 实操：用 SQLite 实现持久化 Agent Loop

先看检查点与状态机的核心实现。SQLite 单文件、事务安全、零运维，足以撑起单机 Agent 的检查点存储。关键在于：**每完成一步，把「新状态」和「这一步的工具结果」放进同一个事务写下去**，保证数据库里不会出现"步数加了、结果没写"这种半截快照。

这里要说清"原子"的边界：事务只覆盖数据库内的两行写入，**工具调用本身（发邮件、下单、改文件）在事务之外**。如果 `act()` 成功了、`save()` 之前进程被杀，副作用已经发生而检查点没记，重启后这一步必然重放。所以光有事务还不够，工具侧必须能按幂等键去重——这是两个机制配着用的，缺一个都不成立。

```python
import json
import sqlite3
import time
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FutureTimeout

SCHEMA = """
CREATE TABLE IF NOT EXISTS checkpoints (
    run_id     TEXT PRIMARY KEY,
    step       INTEGER NOT NULL,
    state      TEXT NOT NULL,      -- 状态机当前状态
    context    TEXT NOT NULL,      -- 压缩后的上下文摘要
    payload    TEXT NOT NULL,      -- 工具结果等中间数据
    updated_at REAL NOT NULL
);
"""

class CheckpointStore:
    def __init__(self, path: str = "agent.db"):
        self.conn = sqlite3.connect(path, check_same_thread=False)
        self.conn.executescript(SCHEMA)
        self.conn.commit()

    def load(self, run_id: str) -> dict | None:
        row = self.conn.execute(
            "SELECT step,state,context,payload FROM checkpoints WHERE run_id=?",
            (run_id,),
        ).fetchone()
        if not row:
            return None
        step, state, context, payload = row
        return {"step": step, "state": state,
                "context": json.loads(context), "payload": json.loads(payload)}

    def save(self, run_id: str, snap: dict) -> None:
        # UPSERT 置于单事务中：步骤推进与落盘要么都成功，要么都不发生
        with self.conn:
            self.conn.execute(
                """INSERT INTO checkpoints(run_id,step,state,context,payload,updated_at)
                   VALUES(?,?,?,?,?,?)
                   ON CONFLICT(run_id) DO UPDATE SET
                     step=excluded.step, state=excluded.state,
                     context=excluded.context, payload=excluded.payload,
                     updated_at=excluded.updated_at""",
                (run_id, snap["step"], snap["state"],
                 json.dumps(snap["context"]), json.dumps(snap["payload"]), time.time()),
            )

# 合法的状态迁移表：任何表外迁移都被视为编程错误
ALLOWED = {"plan": {"act"}, "act": {"observe"},
           "observe": {"plan", "done"}, "done": set()}
_EXECUTOR = ThreadPoolExecutor(max_workers=4)

def act_with_timeout(act, tool, args, timeout: float, idem_key: str):
    """软超时：到点停止等待，但任务不会真的被中止——去重靠 idem_key。"""
    future = _EXECUTOR.submit(act, tool, args, idem_key)
    try:
        return future.result(timeout=timeout)
    except FutureTimeout:
        # cancel() 对已经开始执行的任务返回 False，线程不会被 kill
        future.cancel()
        raise TimeoutError(f"工具 {tool} 超过 {timeout}s 未返回，等待对账")

def run_agent(run_id, store, think, act, max_steps=200, step_timeout=30.0):
    snap = store.load(run_id) or {
        "step": 0, "state": "plan", "context": [], "payload": {}}
    while snap["state"] != "done":
        if snap["step"] >= max_steps:             # 防死循环的硬闸门
            raise RuntimeError(f"超过最大步数 {max_steps}，疑似死循环")
        nxt = think(snap["context"], snap["state"])
        if nxt["to_state"] not in ALLOWED[snap["state"]]:
            raise ValueError(f"非法迁移: {snap['state']} -> {nxt['to_state']}")
        # 幂等键由 run_id + 步号构成：同一步重放多少次，工具侧都只执行一次
        result = act_with_timeout(act, nxt["tool"], nxt["args"], step_timeout,
                                  idem_key=f"{run_id}:{snap['step']}")
        snap["context"].append({"step": snap["step"], "action": nxt, "result": result})
        snap["payload"][str(snap["step"])] = result
        snap["state"], snap["step"] = nxt["to_state"], snap["step"] + 1
        store.save(run_id, snap)                  # 一步一存，崩溃后从此处续跑
    return snap
```

`think` 负责基于当前上下文产出下一步动作，`act` 执行具体工具。注意恢复逻辑：只要传入同一个 `run_id`，`load` 就能取回上次的状态并自然续跑，调用方无需感知「这是第一次还是第 N 次启动」。

这段代码里最容易被当成"已经搞定"的，是 `future.result(timeout=...)`。它只是**不再等**，不是**不再跑**：Python 线程无法被外部强杀，超时那一刻工具仍在后台执行，可能继续写文件、继续发请求；紧接着我们重试这一步，副作用就发生了两次。

要真正"停掉"，只有两条路——进程隔离（`multiprocessing` 或子进程，超时直接 kill），或者给工具自己的客户端设超时（HTTP 的 `timeout=`、数据库的 `statement_timeout`），让它从内部收敛。绝大多数 Agent 框架走第二条，因为进程隔离的序列化与状态同步成本太高。

所以工程上的分工是：**软超时管"等多久"，幂等键管"最多执行一次"**。超时之后不要直接重试，先拿 `idem_key` 去对账——工具侧如果已经执行成功，直接取回结果；确认没执行，再重放。下面这节就是把这件事落到代码里。

## 实操：幂等工具与去重表

接下来把「有副作用」的工具包一层幂等。操作 ID 必须由业务语义稳定派生——同样的输入永远得到同样的 ID，而不能每次调用都随机生成，否则去重无从谈起。

```python
import hashlib

def op_id(*args, **kwargs) -> str:
    """由业务键稳定派生操作 ID：语义相同则 ID 相同。"""
    raw = f"{args}|{sorted(kwargs.items())}"
    return hashlib.sha256(raw.encode()).hexdigest()

class IdempotencyGuard:
    def __init__(self, conn: sqlite3.Connection):
        self.conn = conn
        self.conn.execute("""CREATE TABLE IF NOT EXISTS operations (
            operation_id TEXT PRIMARY KEY, result TEXT, created_at REAL)""")
        self.conn.commit()

    def run_once(self, key: str, fn) -> str:
        row = self.conn.execute(
            "SELECT result FROM operations WHERE operation_id=?", (key,)).fetchone()
        if row:                          # 已成功执行过：回放结果，绝不重放副作用
            return row[0]
        result = fn()                    # 真正产生副作用的操作
        with self.conn:
            self.conn.execute("INSERT OR IGNORE INTO operations VALUES(?,?,?)",
                              (key, result, time.time()))
        return result

guard = IdempotencyGuard(sqlite3.connect("agent.db"))

def publish(post_id: str, content: str) -> str:
    # post_id 是天然的业务幂等键：重试不会重复发帖
    return guard.run_once(op_id("publish", post_id),
                          lambda: http_post("/posts", {"id": post_id, "content": content}))
```

## 实操：可观测性与恢复入口

最后补上结构化日志与 trace。长任务出了问题时，你需要回答「它卡在哪一步、当时状态是什么」，散乱的 `print` 无法支撑这种追溯。用 `trace_id` 串起同一次 run 的所有步骤，日志直接输出 JSON，便于采集与检索。

```python
import logging
from contextvars import ContextVar

trace_id: ContextVar[str] = ContextVar("trace_id", default="-")

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "ts": self.formatTime(record), "level": record.levelname,
            "trace_id": trace_id.get(), "step": getattr(record, "step", None),
            "state": getattr(record, "state", None), "msg": record.getMessage(),
        }, ensure_ascii=False)

logger = logging.getLogger("agent")
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

def main(run_id: str):
    trace_id.set(run_id)                 # 同一次 run 的所有日志共享 trace_id
    store = CheckpointStore("agent.db")
    snap = store.load(run_id)
    logger.info("启动或恢复 Agent", extra={
        "step": snap["step"] if snap else 0,
        "state": snap["state"] if snap else "plan"})
    result = run_agent(run_id, store, think, act)
    logger.info("任务完成", extra={"step": result["step"]})
    return result
```

## 工程实践建议与踩坑

几点来自实战的经验，能帮你绕开常见的坑。

**检查点频率与成本的权衡。** 一步一存最安全，但单步很廉价时频繁落盘的 I/O 开销可观。折中是按「逻辑里程碑」落盘——每完成一次有副作用的调用或每 N 步存一次，接受最多丢失 N 步的代价。选哪档取决于单次重跑成本与落盘成本的比值。

**幂等键要防并发竞态。** 上面 `run_once` 在单进程内没问题，但多进程同时处理同一 `operation_id` 时，两个请求可能都通过了查询、都执行了 `fn()`，只是插入时有一个被 `INSERT OR IGNORE` 丢弃——副作用却已经发生两次。生产环境应改为「先插入占位行（带 `pending` 状态）拿到唯一所有权，再执行副作用，最后更新结果」，用数据库唯一约束做分布式锁。

**给幂等记录设 TTL。** 去重表无限膨胀会拖慢查询。按业务窗口设置过期时间（例如保留 7 天），超过窗口的请求视为新操作。但要注意：TTL 必须长于「最长可能的重试间隔」，否则迟到的重试会穿透去重。

**上下文压缩要外置。** 摘要可以进检查点，但工具返回的大块内容（网页、日志）应落对象存储，检查点只留引用，否则它会膨胀成新的「溢出源」。推荐「近期原文 + 远期摘要」的分层结构。

**恢复前先做状态校验。** 检查点可能因版本升级而结构变化，读取时校验 schema 版本；不兼容的旧检查点要么走迁移，要么明确报错——带病续跑比直接失败更危险。

## 小结

长程 Agent 的可靠性不是靠「让模型更聪明」解决的，而是靠朴素的工程手段：用**检查点**保存可重建的执行现场，用**幂等**把工具变成可安全重试的原语，用**断点续跑**把失败成本从「全量」压缩到「增量」，再用**状态机、步数上限与超时**兜住失控边界，用**结构化日志**留住追溯线索。

当任务从分钟级走向小时级，这些机制不再是锦上添花，而是区分「能演示」与「能上生产」的那条线。先把失败当成默认前提来设计，你的 Agent 才能在真实的异步世界里跑得久、跑得稳。
