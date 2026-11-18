---
title: "一次线上 OOM 的完整复盘"
slug: "production-oom-postmortem"
date: 2026-07-03
draft: false
description: "以一次真实的线上 OOM 为线索，讲透内存分类、现场留存、MAT/pprof 定位与 cgroup 参数调优，并给出复盘模板。"
tags: ["OOM", "JVM", "Go", "故障复盘"]
categories: ["运维部署"]
---

## 事故回放：每隔几个小时就重启一次

某支付服务上线新版本后，Pod 每隔三到四分钟就被重启一次，`kubectl describe` 里赫然写着 `OOMKilled`。研发第一反应是"加内存"，把 limit 从 1G 提到 2G，重启周期延长到了七八分钟——问题依旧，只是死得慢了点。这是处理 OOM 时最典型的误区：把限流阀调大，却不找泄漏的源头。

内存问题的棘手在于它的滞后性：进程是慢慢涨上去的，等到被杀时，真正的"案发时刻"早已过去，现场也随进程消失。所以处理 OOM 的第一原则不是"重启恢复"，而是"先留存现场"。没有现场，后面所有分析都是猜。

## 先分清是哪一类 OOM

OOM 不是一种病，至少要先分清发生在哪一层：

- **JVM 堆 OOM**：`java.lang.OutOfMemoryError: Java heap space`，堆对象撑满 `-Xmx`。
- **元空间/栈 OOM**：`Metaspace`、`unable to create new native thread`，多与类加载、线程数有关。
- **cgroup OOM**：容器 limits 超限，被内核 OOM killer 杀掉，K8s 表现为 OOMKilled。此时 JVM 可能根本没抛出 OOM 异常，因为限制来自容器而非堆参数。
- **系统级 OOM**：宿主机内存耗尽，内核挑选进程杀戮，看 `dmesg` 的 `Killed process`。

判断顺序：先看是容器层还是应用层。`kubectl describe pod` 的 `Last State: Terminated, Reason: OOMKilled` 指向 cgroup 限制；若容器正常但日志里有 `OutOfMemoryError`，则是 JVM 堆问题。两者调优方向完全不同。

## 留存现场：把证据先抓下来

容器被杀前的关键证据要提前布好。JVM 侧加参数自动 dump：

```bash
# JVM 发生 OOM 时自动写出堆转储，并打印 GC 日志
JAVA_OPTS="-XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps/heapdump.hprof \
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags"
```

Go 程序则可暴露 pprof 端点，在内存增长时主动抓取：

```bash
# 程序中引入 net/http/pprof 后，抓堆快照（生效需先 allocs/heap）
go tool pprof -http=:0 http://localhost:6060/debug/pprof/heap

# 对比两个时间点的堆，观察增长对象
go tool pprof -base before.pb.gz http://localhost:6060/debug/pprof/heap

# 内核 OOM 证据：查 cgroup 内存事件与内核日志
cat /sys/fs/cgroup/memory.events        # oom_kill 计数
dmesg -T | grep -i "killed process"
```

还要注意 cgroup 的统计不能完全等同于 JVM 堆：JVM 的堆外内存（直接缓冲、线程栈、Metaspace、JIT 代码缓存）都算进容器 limit 里。设 limit 时必须预留这部分空间，否则 `-Xmx=1G` + `limit=1G` 必然被杀。

## 定位泄漏：MAT 与 pprof

拿到 `.hprof` 后用 Eclipse MAT 打开，重点看 Leak Suspects 报告与 Dominator Tree——它按"持有内存的对象图"排序，能直接找出是谁在累积引用。最常见的泄漏模式是静态集合无界增长、监听器未注销、缓存无淘汰策略。

Go 侧用 `go tool pprof` 的 `top`、`list`、`web` 命令定位分配热点，配合 `inuse_space`（当前占用）与 `alloc_space`（累计分配）区分"还没释放"和"分配太猛"。

本次事故的结论是：新版本引入的一个本地缓存用了 `ConcurrentHashMap` 却没设容量上限，key 是订单号，随流量单调增长。堆 dump 里该类占了 70% 的 retained heap，根因确认。

## cgroup 与运行时参数调优

修复泄漏后，参数也要合理化，给容器留出堆外空间：

```yaml
# K8s：limit 要大于 JVM Xmx，给堆外留 buffer
resources:
  requests:
    memory: "768Mi"
  limits:
    memory: "1Gi"
```

对应 JVM：`-Xmx` 设为 limit 的 60%-70%（如 1Gi limit 设 `-Xmx700m`），剩余给堆外。Go 程序则关注 `GOMEMLIMIT`（Go 1.19+）设置软内存上限，例如 `export GOMEMLIMIT=800MiB`（略低于容器 limit），让 GC 在逼近 limit 前更激进地回收。

## 把 OOM 变成可预警的问题

在事故发生之前，内存问题其实可以提前预警。对 JVM 应采集 GC 日志并用 JMX Exporter 监控堆内存与老年代的增长趋势；对 Go 可用 `runtime/metrics` 暴露 heap 指标。比起"内存用了多少"，"增长速度"是更早的信号——给增长速率设告警，往往能在 OOM 前几十分钟就发现泄漏。压测阶段要做足够长的稳定性测试（soak test），短时压测根本暴露不了缓慢泄漏。把这些监控与测试补齐，OOM 才会从"突发事故"变成"有迹可循的渐进问题"，团队也才有从容应对的底气。

## 复盘方法论与模板

一次合格的复盘不追责、只找系统性问题，产出三块内容：时间线、根因、改进项。可直接套用：

```markdown
# 事故复盘：<服务名> OOM

## 影响
- 时间：2026-07-03 02:14 ~ 03:02
- 范围：支付下单接口，错误率峰值 8%

## 时间线
- 02:14 新版本发布
- 02:31 首次 OOMKilled，Pod 重启
- 02:50 值班介入，误判为内存不足，调大 limit
- 03:02 回滚版本，恢复

## 根因
- 直接原因：无界缓存导致堆内存单调增长
- 根本原因：缓存组件缺少容量评审；上线前未做长稳测试

## 改进项
- [ ] 缓存改为 Caffeine 并设最大容量与过期策略（负责人/日期）
- [ ] limit 与 -Xmx 比例纳入部署模板校验
- [ ] 增加堆内存增长速率告警，提前发现泄漏
```

## 小结

OOM 复盘的关键动作是"先留现场、再分层定位、最后调参兜底"：用堆转储和 pprof 抓住消失的证据，分清 JVM 堆 OOM 与 cgroup OOM 的不同调优路径，把 limit 与运行时参数的关系理顺。而比修复更重要的是复盘——把一次事故转化为缓存规范、参数模板和增长告警这三条可落地的改进。故障不可避免，但同样的故障不该发生两次。
