---
title: "虚拟机莫名卡顿：CPU Steal Time 超标的排查"
slug: "vm-cpu-steal-incident"
date: 2026-04-16
draft: false
description: "一台业务虚拟机响应越来越慢，机内显示还有一半 CPU 空闲。复盘一次由宿主机超卖引发的 steal time 超标，以及我们对 vcpu_quota 的一次长期误解。"
tags: ["KVM", "Steal Time", "超卖", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

4 月中旬的一个周二上午，业务同学反映订单查询接口变慢了，从平时的几十毫秒涨到几百毫秒，偶尔超时。诡异的是，这台服务跑在 KVM 虚拟机里，登上去看 CPU 使用率并不高——用户态 12%，系统态 3%，还有一半以上显示空闲。

按常规思路，CPU 没打满、内存也够，瓶颈应该不在本机。我们顺着查了数据库、缓存、网络，全都没发现异常。这类"指标正常但体感很差"的问题比明显的故障更难查，因为所有常规检查都会告诉你没问题。问题就卡在那里：服务确实慢，但所有"该有的指标"都正常。

## 异常现象

转机出现在一次习惯性的 `vmstat`。它的 CPU 区域最后一列有个平时很少注意的字段 `st`，稳定在 30 左右。再用 `top` 确认，顶部汇总行也写着同样的事实：

```bash
$ nproc
2
$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 184320   9876 3102840    0    0     4    18  412  803 12  3 55  0 30
 3  0      0 183908   9876 3102856    0    0     0    12  398  776 11  4 54  0 31
 2  0      0 183502   9876 3102864    0    0     2    20  405  791 13  3 55  0 29
$ top -b -n 1 | head -n 4
top - 14:36:02 up 41 days,  3:12,  2 users,  load average: 2.31, 2.18, 2.05
Tasks: 287 total,   1 running, 286 sleeping,   0 stopped,   0 zombie
%Cpu(s): 12.3 us,  3.1 sy,  0.0 ni, 54.6 id,  0.0 wa,  0.0 hi,  0.0 si, 30.0 st
```

`30.0 st`。这就是问题所在：虚拟机有 30% 的 CPU 时间不是没活干，而是它想跑、却没被调度上——这段时间宿主机拿去跑别的虚拟机了。在 guest 内部看，这部分时间既不算用户态也不算系统态，只算"被偷走"，所以 `id` 显示空闲，服务却慢得真实。

注意 `load average` 只有 2.31，在一台 2 vCPU 的机器上不算高——因为等待 CPU 的时间根本不进 load，它被记进了 `st`。这就是为什么"所有指标都正常"。

## 排查思路

确认是 steal，下一步找出谁在抢。先看这台 VM 自己的配置：

```bash
$ virsh vcpuinfo order-vm | grep -E "^VCPU|^CPU:|^CPU Affinity"
VCPU:           0
CPU:            3
CPU Affinity:   yyyyyyyy
VCPU:           1
CPU:            4
CPU Affinity:   yyyyyyyy
$ virsh schedinfo order-vm
Scheduler      : posix
cpu_shares     : 1024
vcpu_period    : 100000
vcpu_quota     : -1
emulator_period: 100000
emulator_quota : -1
```

`vcpu_quota = -1`，不限制配额。看起来配置没问题——它本该能拿到足够的 CPU。问题只在宿主机那一侧才说得通。

转到宿主机 `kvm-host-03`：

```bash
$ uptime
 14:41:07 up 63 days,  9:22,  3 users,  load average: 96.42, 91.17, 84.30
$ nproc
32
$ virsh list --all --state-running | tail -n +3 | wc -l
34
$ sar -u 1 1
Linux 3.10.0-1160.el7.x86_64 (kvm-host-03)  04/16/2026  _x86_64_  (32 CPU)
02:41:12 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
02:41:13 PM     all     86.20      0.00      8.91      0.42      0.00      3.40
```

32 个物理核，34 台虚拟机，vCPU 总数 96，超卖比 3:1。load 96 堆在 32 个核上，空闲只剩 3.4%——这不是"某段时间争抢"，这是持续过载。

这里有个细节值得单独说：`sar -u` 里 `%steal` 那一列永远是 **0.00**。因为 steal 是 guest 侧的指标，宿主机自己不被谁"偷"时间，这一列在宿主上毫无意义。我们过去两年一直在这台机器上看 `%steal` 判断有没有超卖问题，看到的永远是 0，于是从来没往这个方向想。真正的观测点必须在 guest 里。

## 处理过程

处理分三步。

第一步应急迁移，把这台业务 VM 和另外几台高优先级虚拟机挪到负载较轻的宿主机，KVM 支持在线迁移，业务基本无感：

```bash
$ virsh migrate --live --persistent order-vm qemu+ssh://kvm-host-07/system
$ virsh list --name --state-running --all | grep order-vm
order-vm
```

第二步是给核心业务真正做资源保障。这里我们踩了一个概念上的坑——第一版方案写的是"给这台 VM 设 `vcpu_quota=80000`，保证它至少拿到 80% 的 CPU"，上线后反而更慢。原因是 `vcpu_quota / vcpu_period` 算出来的是**上限**，不是保底：80000/100000 的意思是整台 VM 每秒最多用 0.8 个核的 CPU 时间，而它有 2 个 vCPU。这不是保障，这是给它上了一道 0.8 核的天花板。

真正能给保底的是 CPU 绑核（cpuset）——把 vCPU 固定到宿主机指定的物理核上，配合 `emulatorpin` 把模拟器线程也绑走，这些核不再被别的 VM 抢占。相对优先级则用 `cpu_shares` 表达：

```bash
$ virsh vcpupin order-vm 0 8
$ virsh vcpupin order-vm 1 9
$ virsh emulatorpin order-vm 10-11
$ virsh schedinfo order-vm --set cpu_shares=4096
Scheduler      : posix
cpu_shares     : 4096
vcpu_period    : 100000
vcpu_quota     : -1
$ virsh vcpuinfo order-vm | grep -E "^VCPU|^CPU:"
VCPU:           0
CPU:            8
VCPU:           1
CPU:            9
```

绑核之后这台 VM 独占两个物理核，`vcpu_quota` 保持 -1（不设上限），避免再出现给自己上枷锁的情况。

第三步，重排整个集群的超卖策略，把单台宿主的 vCPU 总量上限设为物理核数的 1.5 倍，核心业务走独立资源池。

改完观察：

```bash
$ vmstat 1 2 | tail -n 2
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 180112   9876 3102904    0    0     2    14  402  795 13  3 83  0  1
 1  0      0 179840   9876 3102912    0    0     0    16  398  782 12  3 84  0  1
```

`st` 从 30 降到 1，`id` 从 55 升到 84，接口延迟回到正常水平。

## 根因分析

这次故障的根子在资源管理，不在技术本身。

CPU Steal Time 衡量的是虚拟机"可运行但没被调度上"的时间。KVM 里每个 vCPU 都是宿主机上的一个线程，由宿主机调度器统一分配物理时间片。当同一台宿主机上可运行的 vCPU 线程数远超物理核数时，调度器无法满足所有线程，一部分时间就被拿去跑别的 VM——在 guest 内部，这段时间记为 steal。

超分本身是虚拟化的核心价值，但前提是有监控、有约束、有上限。我们的问题有三个：超卖比长期 3:1 且无上限，新 VM 想放哪台就放哪台；没有任何机制保障核心业务；以及监控里从来没有 guest 侧的 `%steal`。宿主机看 CPU 86%、load 96 其实已经说明一切，但这两条曲线在面板上和"业务高峰期"长得太像，没人把它和"某些 VM 被饿着"联系起来。

还有一个观念上的错误，值得写下来：`vcpu_quota` 不是资源保障手段。它是 cgroup 的带宽限制，语义是"最多用多少"，不是"至少给多少"；而且它作用于整台 VM（所有 vCPU 共享这个配额），不是每个 vCPU 各给一份。第一次改配置时把它当保底用，等于给最需要 CPU 的机器又加了一道枷锁——这个错误在改完当天下午就被业务侧的延迟曲线打回来了。

## 复盘总结

最直接的改动是把 steal 纳入监控。所有虚拟机采集 `vmstat` 的 `st`，超过 5% 提醒、10% 告警，并和宿主机的 load、vCPU 总量做关联展示。这条如果早就有，"CPU 没满但服务慢"能第一时间定位。

规范层面，虚拟化集群设了硬性超卖上限：通用业务 1.5:1，核心业务 1:1 或独占物理核；新 VM 上线要登记资源规格，由平台统一分配宿主，不允许随意放置。

技术层面，核心业务一律配 cpuset 绑核 + `emulatorpin` + 提高 `cpu_shares`，把保障写进配置；同时明确一条纪律——`vcpu_quota` 只用于限制"跑飞的机器"，绝不用来表达"保底"。

迁移完成后我们在业务低峰做了压测，确认 `%st` 在负载拉高时也不会反弹，才把这台 VM 交还业务。这件事给我的教训是：虚拟化环境里"CPU 使用率不高"从来不等于"CPU 不是瓶颈"；而当指标和体感矛盾时，别急着否定体感，先去看看那些默认不展示的字段——答案常常就在 `vmstat` 输出的最后一列。
