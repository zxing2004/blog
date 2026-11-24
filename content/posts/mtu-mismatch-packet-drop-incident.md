---
title: "跨机房大包必丢：MTU 不一致导致的诡异故障"
slug: "mtu-mismatch-packet-drop-incident"
date: 2026-07-08
draft: false
description: "7 月业务迁到新机房后，跨机房的大响应体调用必失败、小请求却正常。从一条被防火墙丢掉的 ICMP 报文查起，复盘一次 MTU 不一致引发的丢包。"
tags: ["MTU", "PMTUD", "tcpdump", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

7 月我们做了一次机房迁移，把订单查询和几个依赖服务从老机房搬到了同城的新机房。两边通过一条专线互联，中间跑了一层隧道做二层打通，具体封装由网络团队维护，我们应用层平时不太关心。迁移完成后功能回归一切正常，问题是在第二天下午暴露的：新机房的服务调用老机房的一个报表接口，只要响应体稍微大一点——比如导出一批订单明细——就必然超时失败；而普通的单条查询、健康检查这些小请求，一点事没有。

更让人困惑的是 TLS。新机房侧有几个服务要走老机房的网关做鉴权，TLS 握手偶尔卡住，重试几次又好了。一开始我们把它归为"专线抖动"，毕竟刚迁完，链路不稳似乎也说得过去。但报表接口的失败实在太有规律：小响应成功，大响应必挂，这种非黑即白的表现不像抖动，更像某条明确的边界被越过了。

## 异常现象

我们做的第一组测试是 ping。小包通，大包不通，而且分界点非常清晰。

```bash
# 新机房机器 ping 老机房报表机，禁用分片（DF 位）
$ ping -M do -s 800 10.20.1.40 -c 2
PING 10.20.1.40 (10.20.1.40) 800(828) bytes of data.
808 bytes from 10.20.1.40: icmp_seq=1 ttl=62 time=3.21 ms
808 bytes from 10.20.1.40: icmp_seq=2 ttl=62 time=2.98 ms

$ ping -M do -s 1472 10.20.1.40 -c 2
PING 10.20.1.40 (10.20.1.40) 1472(1500) bytes of data.

--- 10.20.1.40 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1001ms
```

`-M do` 是给 IP 包设置 DF（Don't Fragment）位，`-s 1472` 加上 20 字节 IP 头和 8 字节 ICMP 头正好是 1500。标准以太网 MTU 下这个包刚好能过，现在却 100% 丢。而 800 字节 payload 的包安然无恙。这几乎是在指着 MTU 说：链路上某一段的有效 MTU 比 1500 小。

先看发送侧的出接口。1500 是标准以太网 MTU，也是这台机唯一相关的配置——应用机自己不做隧道封装，隧道口在网关上：

```bash
$ ip link show dev eth0 | grep -i mtu
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP ...
```

也就是说，1500 字节的包从这台机器发出去完全合法，问题出在后面的某一跳。

## 排查思路

先确认发送侧有没有收到"需要分片"的 ICMP 差错：

```bash
$ tcpdump -i any icmp -nn -c 5
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
```

发送端什么都收不到，只看到"包丢了"，于是按原尺寸重传，再丢，再重传。

请网络团队在隧道口 `10.20.1.1` 上抓，顺便看一眼那台网关的接口配置，情况就清楚了：

```bash
$ ip link show dev vxlan100 | grep -i mtu
6: vxlan100: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN ...
$ tcpdump -i any icmp -nn -c 2 host 10.30.5.12
14:22:07.314521 IP 10.20.1.1 > 10.30.5.12: ICMP 10.20.1.40 unreachable - need to frag (mtu 1450), length 556
14:22:08.316043 IP 10.20.1.1 > 10.30.5.12: ICMP 10.20.1.40 unreachable - need to frag (mtu 1450), length 556
```

VXLAN 要在原始二层帧外面再套一层 UDP、IP 和以太网头，大约吃掉 50 字节，所以 underlay 跑 1500 时，隧道里留给上层的载荷只有 1450。我们那个 1500 字节的包进来就超限，网关照规矩回一个 `need to frag (mtu 1450)`——一秒一条，正好是 ping 的默认间隔，节奏和重传完全对得上。

**报文生成没问题，是在回来的路上被拦掉了**。核对新机房出口的防火墙策略，果然有一条"为安全起见"的规则：把发往我们业务网段的 ICMP 全部丢弃，包括 type 3 差错报文。PMTUD 的反馈被这道墙挡在门外，发送端永远收不到"调小 MTU"的提示，只会傻傻按原尺寸重发，于是大包必丢。

TLS 握手卡住是同一个根因的另一种表现形式，而且解释起来更细：服务端返回的 `Certificate` 消息通常有三四个 K，必然被拆成几个满段（发送侧按 eth0 的 1500 算出 MSS 1460），这些 1500 字节的段全部撞在 1450 的隧道口上。而 ClientHello、ServerHello 这些几百字节的报文一路畅通——所以握手总是走到"证书"这一步才停住。所谓"重试几次就好"，更可能与**会话复用**有关：复用的握手不传证书链，自然不被卡；也可能是另一个域名挂的证书链更短，整个消息不超过 1450，一个包就发完了。这种"换个域名就正常"的偶发性最容易误导人，看着像网络抖动。

## 处理过程

处理分三层，先止血，再修根，最后加兜底。第一层，把网关上的隧道口 MTU 从 1450 调到 1400，多留 50 字节余量，先让大包能通。第二层，请网络团队调整防火墙，放行 ICMP type 3 code 4（frag-needed），让 PMTUD 在链路变化时能正常工作。第三层，在出口网关上加 TCP MSS clamp，把 TCP 握手时的 MSS 主动限制到 MTU 减 40（IP 头 20 + TCP 头 20），这样即便遇到不支持 PMTUD 的对端，TCP 流量也不会发出超限的段。

```bash
# 只在网关上调隧道口，eth0（underlay）保持 1500 不动
$ ip link set dev vxlan100 mtu 1400
$ ping -M do -s 1372 10.20.1.40 -c 2
PING 10.20.1.40 (10.20.1.40) 1372(1400) bytes of data.
1400 bytes from 10.20.1.40: icmp_seq=1 ttl=62 time=3.02 ms
1400 bytes from 10.20.1.40: icmp_seq=2 ttl=62 time=2.87 ms
```

`-s 1372` 加上 28 字节的 IP 与 ICMP 头正好 1400，能通就说明新的隧道 MTU 生效了。

这里有个必须守住的约束：**overlay 的 MTU 一定要小于 underlay 减去封装开销**。所以只能降 `vxlan100`，不能把 eth0 也降到 1400——那会变成"内层 1400 + 封装 50 = 1450 > 外层 1400"，照样走不通，甚至更糟（外层还得再分片，而外层通常是 DF 的）。之所以选 1400 而不是刚好 1450，是给将来 underlay 走 PPPoE（1492）或再加一层封装留余量。

第三层的 MSS clamp 要加在**隧道网关**上，不是应用机上——应用机不转发流量，`FORWARD` 链上根本没包经过。这条由网络团队在网关执行：

```bash
# 在隧道网关 10.20.1.1 上执行
$ iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360
$ iptables -t mangle -L FORWARD -n -v | grep TCPMSS
   41  1968 TCPMSS      tcp  --  *      *    0.0.0.0/0  0.0.0.0/0  tcp flags:0x06/0x02 TCPMSS set 1360
```

`1360 = 1400 - 20(IP) - 20(TCP)`。用 `--set-mss` 而不是 `--clamp-mss-to-pmtu` 是刻意的：clamp 只能把 MSS 往下压到路径 MTU，而对端如果压根不发 ICMP，clamp-to-pmtu 也无从得知真实值；显式设定一个确定值，在已知链路参数的情况下更可靠。

改完 ping 大包和报表接口都恢复正常。三层做完，连续观察两天，报表导出和 TLS 握手再没出现过同类问题。

## 根因分析

把链路画出来，根因就清楚了：两家机房之间的专线跑了一层 VXLAN 隧道，封装开销让有效 MTU 从 1500 降到 1450，但迁移方案里只核对了 IP 连通性和带宽，没人把 MTU 当成一项需要对齐的参数。两端的物理接口都还是默认的 1500——它们各自都没错，错在中间那一层没被算进去，于是"名义 MTU 1500"和"实际能过的 1450"之间留了一道谁都看不见的落差。

更关键的是，本该兜底的 PMTUD 被防火墙的 ICMP 策略掐断了。这条策略的出发点不难理解——减少 ICMP 暴露面——但它误伤了 type 3 差错报文，让发送端失去了感知路径 MTU 的能力。于是"隧道缩小 MTU"和"PMTUD 失效"两个本可互相弥补的因素叠加，最终表现为大包必丢、小包正常。TLS 握手只是同一根因在另一种报文尺寸上的投影。

## 复盘总结

这次故障让我们把"MTU"从一个网络团队的专属名词，变成了迁移检查单上的常客。我们定了三条规矩。

第一，凡是涉及跨机房、隧道、Overlay 网络的变更，MTU 必须作为显式参数写进方案，并在割接前用 `ping -M do` 逐档验证大包可达性，而不是默认"1500 全世界通用"。第二，防火墙策略调整要区分 ICMP 类型，放行 frag-needed 这类功能性报文，不能图省事一刀切，并在变更评审里专门过一遍。第三，把链路参数——两端 MTU、封装类型、封装开销、MSS clamp 状态——文档化，网络拓扑变了就同步更新，避免下次迁移又靠抓包现推。

故障那天有位同事说了句挺贴切的话：小包能通，让我们误以为链路是好的；其实链路一直是"半通"的，只是我们没去测那条边界。网络的很多问题都藏在这种"部分可用"里，只有主动去撞那条线，才知道它到底在哪。
