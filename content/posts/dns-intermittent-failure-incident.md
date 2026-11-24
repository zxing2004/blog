---
title: "偶发 DNS 解析失败背后的 conntrack 表溢出"
slug: "dns-intermittent-failure-incident"
date: 2026-03-11
draft: false
description: "业务偶发鉴权失败，日志里是 DNS 解析超时，可手动 dig 又一切正常。顺着一条没人看的 dmesg 记录，挖出一次连接跟踪表被打满的丢包事故。"
tags: ["DNS", "conntrack", "netfilter", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

3 月上旬的一个下午，用户侧开始零星反馈登录转圈。这个问题很诡异：不是全挂，而是每隔一段时间冒出来一次，失败的请求重试一下往往就好了。这类问题最耗人的地方在于，它不会给你一个明确的失败时间点。值班同学盯着监控看了很久，错误率曲线像锯齿一样上下跳，始终没有形成稳定的故障形态。

出问题的服务是鉴权网关，它要访问内网的 `auth.internal` 做令牌校验。最初大家把注意力放在鉴权服务本身，但它的各项指标都很干净——延迟、CPU、连接池占用都没问题。真正的线索藏在应用日志里一行不起眼的报错，以及一个我们平时根本不会去看的地方。

## 异常现象

鉴权网关的日志里反复出现这样的记录，频率不高，一分钟几十条，夹杂在大量正常请求中间。另一类报错更隐蔽，量也更大：上游调用方报"连接超时"，但没有具体错误码，只是 SYN 发出去没有回应。

```text
$ tail -n 3 /data/logs/auth-gateway/app.log
2026-03-06 14:22:07.331 ERROR [auth] verify failed: dial tcp: lookup auth.internal on 10.0.0.2:53: read udp 192.168.1.10:41234->10.0.0.2:53: i/o timeout
2026-03-06 14:22:09.884 ERROR [auth] verify failed: lookup auth.internal: no such host
2026-03-06 14:22:10.102 WARN  [auth] retry succeeded on 2nd attempt
$ dig @10.0.0.2 auth.internal +short
10.12.3.41
```

手动 `dig` 每次都正常返回，几十毫秒。于是第一个判断是"DNS 服务器不稳定"，但管 DNS 的同事同样没看到任何异常查询量或超时。手动测试正常、应用侧失败，这种僵局通常意味着问题不在应用层、也不在服务端，而在两者之间。

## 排查思路

既然怀疑丢包，最直接的办法是抓包。我们在网关这台机器上抓 53 端口的 UDP：

```bash
$ tcpdump -i any -nn port 53 -c 6
14:23:11.203401 IP 192.168.1.10.41234 > 10.0.0.2.53: 38211+ A? auth.internal. (31)
14:23:11.203429 IP 192.168.1.10:41235 > 10.0.0.2.53: 51093+ A? auth.internal. (31)
14:23:11.203450 IP 192.168.1.10.41236 > 10.0.0.2.53: 12003+ A? auth.internal. (31)
14:23:12.204188 IP 192.168.1.10:41234 > 10.0.0.2.53: 38211+ A? auth.internal. (31)
```

连续几个查询只有请求、没有回应，客户端在超时后重发。包确实从机器上发出去了，DNS 服务端也说没收到——问题要么在中间链路，要么在本机内核的丢弃逻辑里。注意报错里"no such host"和"i/o timeout"是两种：前者是收到了 `SERVFAIL` 或 `REFUSED`，后者是干脆没回应。两种混在一起，说明不是 DNS 服务端在拒绝，而是**一部分包根本没走到**。

我们这台机器装了 Docker，iptables 有 NAT 规则，意味着它会走连接跟踪。于是去看一个平时没人看的地方——内核自己记录的网络统计：

```bash
$ sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_count = 1048588
net.netfilter.nf_conntrack_max = 1048576
$ dmesg -T | grep -i conntrack | tail -n 3
[Wed Mar  6 14:19:02 2026] nf_conntrack: table full, dropping packet
[Wed Mar  6 14:19:04 2026] nf_conntrack: table full, dropping packet
[Wed Mar  6 14:19:05 2026] nf_conntrack: table full, dropping packet
$ head -2 /proc/net/stat/nf_conntrack
entries  searched found new invalid ignore delete delete_list insert insert_failed drop early_drop icmp_error
0000a3f1 2f19c8e2 0001f2a4 0000d1c7 00000000 00000000 0000b93e 00000000 0001a2b1 0000094c 00000412 00000012 00000000
```

`count` 已经把 `max` 顶穿了（内核允许极小的溢出余量），`dmesg` 里 `table full, dropping packet` 在刷。连接跟踪表满了，内核对**新连接的第一个包**直接丢弃——不回报错、不返回码，就是静默消失。

这也解释了两类报错的来源：UDP DNS 查询的新包被丢，表现为 `i/o timeout`；TCP 新建连接（上游调用方的请求）的 SYN 被丢，表现为连接超时，客户端重试一次可能刚好赶上表里有空隙，所以"重试就好"。手动 `dig` 成功，纯粹是因为它执行的那一瞬间刚好撞上了空闲窗口——偶发故障里，"我这儿测试正常"是最不可靠的证据。

## 处理过程

应急第一步是把上限抬高，给系统留出余量：

```bash
$ sysctl -w net.netfilter.nf_conntrack_max=2097152
net.netfilter.nf_conntrack_max = 2097152
$ sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_count = 1048631
net.netfilter.nf_conntrack_max = 2097152
```

注意 `count` 并不会因为上限变大而下降——放宽阈值只是让新连接不再被拒，已经在表里的条目该怎么超时还怎么超时。真正让它降下去的是后面两步。

这里还踩了一个坑。我一开始想顺手把哈希桶也扩了，照着网上抄了句 `sysctl -w net.netfilter.nf_conntrack_buckets=524288`，直接报只读：

```bash
$ sysctl -w net.netfilter.nf_conntrack_buckets=524288
sysctl: cannot stat /proc/sys/net/netfilter/nf_conntrack_buckets: Permission denied
$ cat /sys/module/nf_conntrack/parameters/hashsize
262144
```

`nf_conntrack_buckets` 这个 sysctl 在运行时是只读的，要改桶数只能写模块参数 `hashsize`，或者在内核启动参数里带 `nf_conntrack.hashsize=`：

```bash
$ echo 524288 > /sys/module/nf_conntrack/parameters/hashsize
$ cat /sys/module/nf_conntrack/parameters/hashsize
524288
```

经验比例是 `hashsize ≈ nf_conntrack_max / 4`。只加 `max` 不加桶，表项会挤在长链表里，查找退化成线性扫描——上限是够了，延迟上来了。

应急之后错误率立刻下降，但 `count` 很快又爬到一百万以上，说明背后有持续制造连接的来源。按源 IP 和目标聚合看连接状态：

```bash
$ conntrack -L 2>/dev/null | awk '{print $4}' | sort | uniq -c | sort -rn | head -3
  612034 ESTABLISHED
  318922 TIME_WAIT
  102411 SYN_SENT
```

`TIME_WAIT` 和 `SYN_SENT` 加起来四十多万，这是短连接的典型特征。继续按源地址聚合，很快锁定一个上游服务：它的 HTTP 客户端没开 keep-alive，每个请求都重新建连、握手、用完就关。压测和晚高峰时每秒新建数千连接，每条连接在 `TIME_WAIT` 里要躺两分钟（`nf_conntrack_tcp_timeout_time_wait` 默认 120 秒），累积起来就是几十万条。

治本分两路：一是推动那个服务开连接池、复用长连接，把新建速率压下来；二是给这个网关单独调低几个超时参数，让短连接的表项回收得更快：

```bash
$ sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
$ sysctl -w net.netfilter.nf_conntrack_tcp_timeout_syn_sent=30
```

两件事落地后，`count` 稳定在二十万左右，偶发解析失败彻底消失。

## 根因分析

表面看是 DNS 问题，本质是连接跟踪资源耗尽。

`nf_conntrack` 是 netfilter 用来记录每条连接状态的内核结构，条目数由 `nf_conntrack_max` 控制。它有个特点：连接关闭后条目不会立刻消失，要等超时——`ESTABLISHED` 默认五天，`TIME_WAIT` 默认两分钟，`SYN_SENT` 默认两分钟。大量短连接在短时间内反复创建，表项来不及回收，就会迅速逼近上限。

一旦表满，内核的处理是**直接丢弃新连接的首包**，而且不产生任何应用层可见的错误码。于是同一个根因在不同协议上表现为不同症状：DNS 是 UDP，没有重传保障，表现成解析超时或 `no such host`；TCP 的 SYN 被丢，表现成连接超时。这也是为什么我们一开始被"DNS 故障"这个表象带偏——它只是最先喊疼的那个。

三个因素叠加才酿成这次事故：某个上游服务短连接激增是导火索；`nf_conntrack_max` 沿用默认值、和这台机器的实际连接量不匹配是放大器；完全没有对 conntrack 使用量做监控和告警，是真正的盲区。前两个是技术债，最后一个是管理债。

还有一个容易被忽略的点：conntrack 表是**按 netns 独立**的。如果问题出在某个容器里，在宿主机上 `sysctl` 读到的 count 会是干净的，只能靠 `dmesg` 的时间点去对。这次我们运气好，打满的是宿主机的初始 netns。

## 复盘总结

这次之后，我们把 conntrack 纳入了常规监控：采集 `nf_conntrack_count / nf_conntrack_max` 的比值，70% 提醒、85% 告警；更关键的是盯 `/proc/net/stat/nf_conntrack` 里的 `drop` 和 `insert_failed` 计数，只要非零增长就立刻告警——因为 `count/max` 看着还没满，丢包可能已经在发生了。同时把 `dmesg` 的 `table full` 关键字接进了日志告警。

应用侧补了两条规范：所有内部 HTTP 客户端必须开连接复用，禁止"一次请求一条连接"；关键服务在本机部署 dnsmasq 做本地缓存，即使上游解析偶发失败也能顶一阵。参数层面，把 `nf_conntrack_max` 和 `hashsize` 按机器规格写进了初始化脚本，不再依赖默认值。

最后一点体会：排查这类问题最容易掉进的坑，是"我测试是正常的，所以没问题"。当现象是偶发的时候，手动测试的价值很有限，真正有用的是把那一秒的内核计数、抓包结果和应用日志对齐。这次如果不是顺手看了一眼 `dmesg`，我们大概还在怀疑 DNS 服务器。
