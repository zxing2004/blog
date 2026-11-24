---
title: "一次 Kubernetes 节点 NotReady 的完整复盘"
slug: "k8s-node-notready-incident"
date: 2026-01-15
draft: false
description: "凌晨一个业务节点突然 NotReady、Pod 被持续驱逐。日志报磁盘没空间，df 却显示还有三分之一空闲——真正的指标是 inode，而那条线索二十分钟前就已经亮在面板上了。"
tags: ["Kubernetes", "kubelet", "inode", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

1 月中旬的一个周三凌晨两点十分，值班手机响了。告警来自我们自建的 Kubernetes 集群，一台承载订单查询业务的 worker 节点从 Ready 翻转成 NotReady，上面十几个 Pod 的流量被迫挤到剩下两台节点上。

这套集群跑了将近两年，节点都是物理机，系统盘和 kubelet 数据盘是同一块 200G 的 SSD。平时凌晨流量很低，发布也都避开这个时段，所以告警来得毫无征兆。集群没有专职值班 SRE，我和另一位同事轮班。凌晨的 NotReady 大多是无害抖动，所以第一反应是"先恢复业务，再查原因"。事后看，真正拖长故障时间的不是技术难度，而是我们看见了一条关键线索，却按经验把它判成了误报。

## 异常现象

登上跳板机第一件事是看节点状态。`kubectl get nodes` 里 worker-07 已经是 NotReady。`describe` 拉出来的 Conditions 里，有一个我们当时没当回事的细节：

```bash
$ kubectl describe node worker-07 | sed -n '/Conditions:/,/Addresses:/p'
Conditions:
  Type                 Status  LastHeartbeatTime           LastTransitionTime          Reason                      Message
  ----                 ------  -------------------         ------------------          ------                      -------
  MemoryPressure       False   Wed, 15 Jan 2026 02:09:41   Tue, 14 Jan 2026 22:14:03   KubeletHasSufficientMemory  kubelet has sufficient memory available
  DiskPressure         True    Wed, 15 Jan 2026 02:09:41   Wed, 15 Jan 2026 01:58:12   KubeletHasDiskPressure      kubelet has disk pressure
  Ready                Unknown Wed, 15 Jan 2026 02:09:41   Wed, 15 Jan 2026 02:10:21   NodeStatusUnknown           Kubelet stopped posting node status
```

`DiskPressure` 是 **True**，而且从 01:58:12 就开始了——比心跳停摆早了十二分钟。`LastHeartbeatTime` 全部停在 02:09:41，说明 kubelet 最后一次成功上报是那个时刻，之后再也没有更新过（Ready 变 Unknown 是控制面等了默认 40 秒的宽限期之后自己判的）。

也就是说，这不是一次突如其来的故障，而是一次持续了十二分钟、逐步恶化的过程：01:58 节点进入磁盘压力，kubelet 开始按硬驱逐策略杀 Pod；02:09 kubelet 连节点状态都写不进去了，心跳断掉。凌晨业务量低，前面那批驱逐只让监控上多了几条零星的错误率毛刺，没人被叫醒——因为我们只给 NotReady 配了告警，**没给 DiskPressure 配**。

## 排查思路

心跳停止，常见原因无非三类：网络不通、kubelet 进程挂了、节点资源耗尽到 kubelet 无法工作。先 ping，通；SSH 能登；`systemctl status kubelet` 显示 `Active: active (running)`，进程还在，已经连续跑了近四个小时。进程活着但不再工作，方向就收到"资源耗尽"上。

接着看 journalctl：

```bash
$ journalctl -u kubelet --since "2026-01-15 02:00:00" --no-pager | grep -iE "no space|disk pressure|evict" | tail -n 3
Jan 15 02:09:52 worker-07 kubelet[2187]: E0115 02:09:52.114903 2187 kubelet_node_status.go:452] Error updating node status, will retry: write /var/lib/kubelet/pods/.../volumes: no space left on device
Jan 15 02:09:56 worker-07 kubelet[2187]: W0115 02:09:56.329010 2187 kubelet.go:1452] skipping pod synchronization - kubelet is unhealthy
```

`no space left on device`。第一反应当然是数据盘满了，跳上去 `df -h`：

```bash
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       197G  121G   67G  65% /
```

空间还剩三分之一。看到这个结果，我们当场把磁盘排除了——"kubelet 报没空间，可盘只用了 65%，那大概是它自己的日志误报"。这一步是整个排查里最贵的一次误判：**`ENOSPC` 不止意味着空间不够，也意味着 inode 不够**。空间和 inode 是两个独立的水位，`df -h` 只看前者。

补一条 `df -i`，问题当场现形：

```bash
$ df -i /
Filesystem       Inodes    IUsed    IFree IUse% Mounted on
/dev/sda2      13107200 13106992      208  100% /
$ du -xhd1 /var/log 2>/dev/null | sort -h | tail -3
 61G    /var/log/pods
3.2G    /var/log/journal
512M    /var/log/messages
$ find /var/log/pods -type f | wc -l
11794322
```

一千三百万个 inode 只剩两百零八个。`/var/log/pods` 一个目录占了 61G、一千一百多万个文件，平均每个五 K 出头——全是容器标准输出的 json 日志，没有轮转、没有大小上限。

## 处理过程

先停服务避免边清边写，再按时间删日志，最后把上限补上。

```bash
$ systemctl stop kubelet && systemctl stop containerd
$ find /var/log/pods -type f -name '*.log' -mtime +1 -delete
$ crictl rmi --prune
Deleted: 3a9f...c21e
Deleted: 7b40...e93a
$ df -i /
Filesystem       Inodes    IUsed    IFree IUse% Mounted on
/dev/sda2      13107200  6894114  6213086   53% /
$ systemctl start containerd && systemctl start kubelet
$ kubectl get nodes worker-07
NAME        STATUS   ROLES    AGE    VERSION
worker-07   Ready    <none>   712d   v1.28.6
```

删掉一天前的日志、清掉悬空镜像层，inode 从 100% 回到 53%。心跳在半分钟内恢复，被驱逐的 Pod 由 ReplicaSet 重新调度回集群。

但只清不够。这批日志能长到这个规模，是因为 kubelet 的容器日志上限从来没配过。默认是每个容器 `--container-log-max-size=10Mi`、`--container-log-max-files=5`，而这套集群是手写 systemd unit 启动的，参数在迁移时漏掉了，等于允许每个容器无限写。补进 drop-in 后逐台滚动重启：

```ini
# /etc/systemd/system/kubelet.service.d/10-log.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--container-log-max-size=50Mi --container-log-max-files=4"
```

最后把 `/var/log` 单独挂了一块盘，跟 `/` 解耦——日志再疯，也吃不掉 nodefs 的 inode 池。

## 根因分析

直接原因是 inode 耗尽：容器日志无上限地写，一千多万个小文件把 ext4 的 inode 表吃干。kubelet 更新节点状态时要给 Pod 卷目录写临时文件，内核返回 `ENOSPC`，心跳停摆，节点 NotReady。

但真正值得写的根因有两层，都在我们身上。

第一层，**我们把 `ENOSPC` 等同于"磁盘满"**。这是 Linux 上非常常见的认知捷径，而 inode 是它的盲区。`df -h` 是每个人下意识敲的第一条命令，`df -i` 却几乎没人敲——除非你已经想到 inode。

第二层，也是更贵的：**DiskPressure 明明亮了十二分钟，既没被告警，也没被我们看见**。kubelet 的默认硬驱逐阈值里就包含 `nodefs.inodesFree<5%`（Linux 节点），所以 inode 掉到 5% 以下时 DiskPressure 会先变 True、先开始驱逐 Pod，然后才是 kubelet 写不进状态、节点 NotReady。也就是说这套系统提前十二分钟就喊了"inode 要满了"，我们只是没把这个信号接进告警通道。

完整链条是：日志无上限（设计缺陷）→ inode 耗尽（资源耗尽）→ DiskPressure 告警缺失（监控缺口）→ kubelet 无法写状态（服务失效）→ 节点 NotReady（业务影响）。链条上有两个可以掐断的点，我们一个都没设。

## 复盘总结

恢复之后我们固化了四条。

第一，inode 和空间平级监控。node_exporter 的 `node_filesystem_files_free` 直接能算使用率，80% 提醒、90% 严重，和 `node_filesystem_avail_bytes` 同级。

第二，**DiskPressure / MemoryPressure / PIDPressure 三种节点状况全部接入告警**，不再只盯 NotReady。这条如果早就有，故障在 01:58 就会被拦住，根本走不到 NotReady。

第三，全集群统一容器日志上限，写进 kubelet 的 systemd drop-in，新节点上线作为基线检查项；`/var/log` 独立分区，避免和应用数据抢同一个 inode 池。

第四，做了一个节点体检脚本，把 `df -h`、`df -i`、悬空镜像计数、kubelet 心跳状态打包成一条命令，交接班跑一遍。

我们后来还补了一条规矩：**看到 `ENOSPC` 先敲 `df -i`**，写在 runbook 第一页。当晚真正省时间的不是任何复杂技术，而是一条几乎没人会主动敲的命令——以及一个亮了十二分钟、没人接的信号。
