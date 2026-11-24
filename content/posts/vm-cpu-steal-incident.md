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

4 月中旬的一个周四上午，业务同学反映订单查询接口变慢了，从平时的几十毫秒涨到几百毫秒，偶尔超时。诡异的是，这台服务跑在 KVM 虚拟机里，登上去看 CPU 使用率并不高——用户态 12%，系统态 3%，还有一半以上显示空闲。

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
top - 14:36:02 up 41 days,  3:12,  2 users,  load average: 0.92, 0.88, 0.85
Tasks: 287 total,   1 running, 286 sleeping,   0 stopped,   0 zombie
%Cpu(s): 12.3 us,  3.1 sy,  0.0 ni, 54.6 id,  0.0 wa,  0.0 hi,  0.0 si, 30.0 st
```

`30.0 st`。这就是问题所在：这台 VM 平均只有 45% 的时间真的需要 CPU（`us + sy + st`≈45），可这 45% 里有 30 个百分点没轮到它跑——宿主机拿去跑别的虚拟机了。在 guest 内部，这段时间既不算用户态也不算系统态，只记成"被偷走"。所以 `id` 那 55% 是诚实的空闲，不是余量：**余量从一开始就不存在**。

再看 `load average` 0.92——一台 2 vCPU 的机器，连半个核都没跑满，任谁看都是"很闲"。问题在于 load 只回答"有多少任务在排队"，不回答"排了多久"。vCPU 被抢走的那 30% 时间里，任务确实排在队列上，可队列永远只有一两个，平均下来 load 自然不会高；而每个请求实际等到的调度间隔被拉长到几十毫秒，延迟就从几十毫秒涨到了几百毫秒。**load 低只能说明队短，不能说明队快。**

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
02:41:13 PM     all     87.27      0.00      8.91      0.42      0.00      3.40
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

真正能给保底的是 CPU 绑核（cpuset）——把 vCPU 的落点固定到宿主机指定的物理核上，再用 `emulatorpin` 把模拟器线程也绑走，减少它们跟 vCPU 抢同一批核。相对优先级则用 `cpu_shares` 表达：

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

这里得说清楚 `vcpupin` 到底保证了什么：它约束的是**这台 VM 的 vCPU 允许落在哪些物理核上**，是亲和性，不是排他锁。别的 VM 的 vCPU 照样可以被调度到 8、9 号核。要真正做到独占，还得在宿主机上把这些核从通用调度里摘出去（内核参数 `isolcpus` / cgroup cpuset），或者把同宿主其他 VM 的 vCPU 显式排开。我们后来走的是后者——配合第三步的资源池划分，让这两台宿主上的其他 VM 不再使用 8–11 号核。

`vcpu_quota` 保持 -1（不设上限），避免再出现给自己上枷锁的情况。

第三步，重排整个集群的超卖策略，把单台宿主的 vCPU 总量上限设为物理核数的 1.5 倍，核心业务走独立资源池。

改完观察：

```bash
$ vmstat 1 2 | tail -n 2
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 180112   9876 3102904    0  0     2    14  402  795 41  4 54  0  1
 1  0      0 179840   9876 3102912    0  0     0    16  398  782 42  4 53  0  1
```

`st` 从 30 降到 1，`us + sy` 从 15 涨到 45，而 `id` 几乎没动（55 → 54）。这三个数字放在一起才把话说完整：那 30% 从来不是空闲，一直是业务要用、但拿不到的 CPU。以前它被记在 `st` 里，看起来像"机器很闲"；现在它回到了 `us`/`sy`，延迟也跟着回到了几十毫秒。

## 根因分析

这次故障的根子在资源管理，不在技术本身。

CPU Steal Time 衡量的是虚拟机"可运行但没被调度上"的时间。KVM 里每个 vCPU 都是宿主机上的一个线程，由宿主机调度器统一分配物理时间片。当同一台宿主机上可运行的 vCPU 线程数远超物理核数时，调度器无法满足所有线程，一部分时间就被拿去跑别的 VM——在 guest 内部，这段时间记为 steal。

超分本身是虚拟化的核心价值，但前提是有监控、有约束、有上限。我们的问题有三个：超卖比长期 3:1 且无上限，新 VM 想放哪台就放哪台；没有任何机制保障核心业务；以及监控里从来没有 guest 侧的 `%steal`。宿主机看 CPU 87%、load 96 其实已经说明一切，但这两条曲线在面板上和"业务高峰期"长得太像，没人把它和"某些 VM 被饿着"联系起来。

还有一个观念上的错误，值得写下来：`vcpu_quota` 不是资源保障手段。它是 cgroup 的带宽限制，语义是"最多用多少"，不是"至少给多少"；而且它作用于整台 VM（所有 vCPU 共享这个配额），不是每个 vCPU 各给一份。第一次改配置时把它当保底用，等于给最需要 CPU 的机器又加了一道枷锁——这个错误在改完当天下午就被业务侧的延迟曲线打回来了。

## 复盘总结

最直接的改动是把 steal 纳入监控。所有虚拟机采集 `vmstat` 的 `st`，超过 5% 提醒、10% 告警，并和宿主机的 load、vCPU 总量做关联展示。这条如果早就有，"CPU 没满但服务慢"能第一时间定位。

规范层面，虚拟化集群设了硬性超卖上限：通用业务 1.5:1，核心业务 1:1 或独占物理核；新 VM 上线要登记资源规格，由平台统一分配宿主，不允许随意放置。

技术层面，核心业务一律配 cpuset 绑核 + `emulatorpin` + 提高 `cpu_shares`，把保障写进配置；同时明确一条纪律——`vcpu_quota` 只用于限制"跑飞的机器"，绝不用来表达"保底"。

迁移完成后我们在业务低峰做了压测，确认 `%st` 在负载拉高时也不会反弹，才把这台 VM 交还业务。这件事给我的教训是：虚拟化环境里"CPU 使用率不高"从来不等于"CPU 不是瓶颈"；而当指标和体感矛盾时，别急着否定体感，先去看看那些默认不展示的字段——答案常常就在 `vmstat` 输出的最后一列。
