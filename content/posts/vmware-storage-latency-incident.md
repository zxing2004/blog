---
title: "VMware 存储延迟飙升：一次快照失控的教训"
slug: "vmware-storage-latency-incident"
date: 2026-09-09
draft: false
description: "9 月一批虚拟机同时变慢，数据库写入卡顿。vCenter 上的延迟曲线一路走高，最后在一台机器上找到了一个忘了删的、滚了好几天的快照。"
tags: ["VMware", "快照", "存储性能", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

9 月上旬的一个上午，监控陆续报起应用超时，最先喊疼的是订单库——写入延迟突然拉长，前端下单开始卡。同一时间段，另外几台跑批量任务的虚拟机也变得奇慢，平时几分钟的跑批拖到半小时还没结束。这些机器分布在不同的业务组，唯一的共同点是都在同一个 vSphere 集群、挂着同一个 datastore。

这套虚拟化环境跑了我们大部分业务，平时性能很稳。数据库那台是给业务写日志和订单的，对 IO 延迟极其敏感。我们最初怀疑是存储阵列老化或者某台宿主机出了问题，因为"一批虚拟机同时变慢"听起来太像共享存储的锅。处理原则是先把最要紧的数据库救回来，再慢慢查源头。回头看，真正的线索一直躺在 vCenter 的性能图表里，只是我们花了一段时间才把注意力从"阵列是不是坏了"挪到"是不是谁把存储用坏了"。

## 异常现象

登上数据库虚机，第一件事是看它自己的磁盘表现。`iostat -x` 连着打几秒，await 高得离谱。

```text
$ iostat -x 1 3
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           12.40    0.00    8.91   38.72    0.03   39.94

Device            r/s     w/s     rkB/s    wkB/s  avgqu-sz   await  r_await  w_await  %util
sda              14.20  122.60   568.00  3910.40    18.74   137.21     6.83   157.94   99.80
sda              12.80  131.40   512.00  4208.00    21.03   152.66     7.11   174.02   99.90
```

正常时候这块盘的 await 是个位数毫秒，现在写 await 稳定在 150 毫秒以上，`%util` 顶到 100%，`iowait` 也飙到近 40%。这不是应用慢，是底层存储回应得太慢。跨到 vCenter 看 datastore 的延迟曲线，同一时间点确实从几毫秒抬到了两位数，跟虚机里的现象对得上。

## 排查思路

我们进入 ESXi 宿主机，用 esxtop 看磁盘设备视图（`esxtop` 后按 `u`；按 `d` 是 adapter 视图，只有 vmhba 粒度，看不到具体的 `naa.` 设备），确认延迟发生在哪一层，并顺手看了这个 datastore 的容量。

```text
esxtop - 磁盘视图（关键列）
Device       DAVG/cmd    KAVG/cmd    GAVG/cmd    QAVG    CMDS/s
naa.6000...    182.34       3.21      185.55    12.08    842.60

$ df -h /vmfs/volumes/datastore1
Filesystem            Size  Used Avail Use% Mounted on
/vmfs/volumes/datastore1  2.0T  1.88T  120G  94% /vmfs/volumes/datastore1
```

`DAVG/cmd`（设备延迟）是 182 毫秒，而 `KAVG/cmd`（内核队列延迟）只有几毫秒——延迟基本全在存储设备侧，不是宿主机队列堵了。这把方向进一步收窄到 datastore 本身。

使用率 94%，已经逼近警戒线。但 120G 可用空间理论上不算致命，除非有东西在疯狂吞噬剩余空间并放大写入。我们开始逐台看虚机的磁盘文件，很快在一台跑报表服务的虚机上发现了异常——它的目录里有一串不断膨胀的 delta 文件。

## 处理过程

登上那台 ESXi 宿主机，先用 vim-cmd 定位虚机和它的快照树，再看它实际占了多少空间。

```bash
$ vim-cmd vmsvc/getallvms | grep -i report
23     report-prod   [datastore1] report-prod/report-prod.vmx   ubuntu64Guest
$ vim-cmd vmsvc/snapshot.get 23
--ROOT Snapshot--
	Snapshot Name: VEEAM BACKUP TEMPORARY SNAPSHOT
	Snapshot Id: 19
	Snapshot Description: Created by Veeam Backup & Replication
	Snapshot Creation Time: 2026-09-05 02:00:13
$ ls -lh /vmfs/volumes/datastore1/report-prod/*.vmdk
-rw-------  1 root  root  198G Sep  9 09:12 report-prod-000001-delta.vmdk
-rw-------  1 root  root   628 Sep  9 09:12 report-prod-000001.vmdk
-rw-------  1 root  root  240G Sep  9 09:12 report-prod-flat.vmdk
-rw-------  1 root  root   628 Sep  9 09:12 report-prod.vmdk
```

一个快照，名字直接把凶手写出来了：`VEEAM BACKUP TEMPORARY SNAPSHOT`，创建时间 9 月 5 日凌晨两点——正好是每晚备份任务的时间点。理清这几行文件是看懂 ESXi 快照的关键：`report-prod.vmdk` 和 `report-prod-000001.vmdk` 都只有 628 字节，是**描述符**（纯文本，记着对应数据文件的路径与参数）；真正装数据的是 `-flat.vmdk`（基础盘的 240G）和 `-000001-delta.vmdk`（快照期间的写入增量）。所以涨到 198G 的是 delta，不是那个同名的描述符——在 `ls` 里它们名字只差一个 `-delta` 后缀，看错一次就会得出"基础盘被改了"的错误结论。

根因至此基本确认：备份任务在 9 月 5 日给这台虚机打了快照，备份中途失败，快照既没被提交也没被删除，就这么挂了四天。这四天里虚机的所有写入都不再落到基础盘，而是追加到 delta 上——报表服务每天要重算历史数据，写入量本来就大，198G 就是这么堆出来的。

性能上的伤害来自两处叠加。一是 delta 持续增长：VMFS 要不断为这个稀疏文件分配新块、更新元数据，同时和同 datastore 上其他虚机的 IO 争抢文件锁；delta 越大，这部分开销越重。二是 datastore 已经被撑到 94%——VMFS 在接近写满时分配新块的路径更长，碎片也更严重。两件事互为因果：delta 涨得快，把 datastore 撑满；datastore 越满，delta 的写入延迟越高。同一 datastore 上的数据库和跑批虚机，就是这么被连坐拖慢的。

处理顺序我们很谨慎：先确认那个备份任务已经有别的方式完成了数据保护，再删快照。直接删快照会触发 delta 合并回基础盘，期间 IO 压力会短暂上升，所以放在业务低峰做。

```bash
$ vim-cmd vmsvc/snapshot.remove 23 19
# 删除请求是异步的，合并任务在后台执行，约 20 分钟完成
$ ls -lh /vmfs/volumes/datastore1/report-prod/*.vmdk
-rw-------  1 root  root  240G Sep  9 11:42 report-prod-flat.vmdk
-rw-------  1 root  root   628 Sep  9 11:42 report-prod.vmdk
$ df -h /vmfs/volumes/datastore1
Filesystem            Size  Used  Avail Use% Mounted on
/vmfs/volumes/datastore1  2.0T  1.68T  320G  84% /vmfs/volumes/datastore1
```

合并完成后可用空间从 120G 回到 320G，释放的就是那 198G delta。datastore 延迟和数据库虚机的 await 在半小时内逐步回落到正常区间，应用超时告警停止。

## 根因分析

这件事的直接触发点是一次失败的备份，但让它演变成全集群故障的，是三个缺口叠加。

第一，备份失败后没有回滚快照。备份任务打快照、读数据、删快照，是一条完整流程；它在中途失败后，清理步骤没执行，快照就那样挂着。而我们的备份系统对"失败后残留快照"这件事既不告警也不自愈。第二，缺少快照生命周期管理。快照本是临时状态，我们却没有"最多保留多久"的约束，一个四天前的快照谁也没想起来去看一眼。快照一挂就是四天，delta 从零滚到 198G，性能劣化是时间累积出来的——这也是快照类故障最典型的特征：它不是突然坏的，是慢慢坏透的。第三，存储容量告警缺位。datastore 到 94% 才被我们发现，而且是顺手查到的，不是监控告诉我们的——如果 80% 就告警，我们会在 delta 文件刚开始膨胀时介入。

## 复盘总结

恢复之后我们补了三道防线。第一，给所有虚机加快照存活时间限制，超过 24 小时的快照自动告警、超过 72 小时强制进入处理队列；备份任务改成 finally 式的清理逻辑，无论成功失败都要尝试删快照，删不掉就告警。第二，备份失败单独设一条高优先级告警，不再混在普通任务日志里。第三，datastore 水位纳入监控，80% 告警、90% 严重，并把"delta 文件总大小"作为一个独立指标盯住。

这次故障教会我们一件事：虚拟化环境里，很多"存储性能问题"的根源其实是一个被遗忘的临时状态。快照、克隆、悬挂的虚拟磁盘，这些东西平时无害，一旦没人管，就会在你看不见的地方慢慢长大，直到有一天把整条链上的业务一起拖慢。把临时状态当正式资产来管，才不会重蹈覆辙。
