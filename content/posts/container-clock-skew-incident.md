---
title: "容器里的时间漂移：一次令牌校验失败的追查"
slug: "container-clock-skew-incident"
date: 2026-08-13
draft: false
description: "8 月一个服务突然大量报鉴权失败，令牌一签发就过期。证书查了三遍都没问题，最后在容器的环境变量里找到了那口被调快了四分钟的钟。"
tags: ["时间同步", "chrony", "TLS", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

8 月中旬，一个负责对接外部支付网关的服务开始大量报鉴权失败。这条调用链平时很稳，突然之间成功率从接近 100% 掉到七成左右，日志里全是"令牌已过期""时间戳校验不通过"这类报错。诡异的是，同一个集群里另外几个也调这家网关的服务完全正常，网关那边也坚称接口和签名规则都没动过。

"只有一家不对"的故障，十有八九是配置差异。这个服务刚好是几个服务里唯一一个用了自定义基础镜像的——它为了跑一个历史数据回放的测试逻辑，镜像里额外装过一些东西。这个细节当时只是模糊地闪过脑海，没人当回事。我们按最可能的路径查起：先确认对方证书和签名规则没问题。

## 异常现象

从报错侧看，最刺眼的是两类消息混在一起：

```text
2026-08-13 14:21:03.512 ERROR [pay] create order failed: token expired, exp=1786611663, now=1786611907
2026-08-13 14:21:07.884 WARN  [pay] signature rejected: timestamp out of tolerance, skew=247s
```

第一类是 JWT 过期。`exp` 是签发时服务端写进去的过期时间，`now` 是本地时间——注意这两行日志里，同一个请求的 `now` 比 `exp` 大了 244 秒。而网关签发的令牌有效期只有 **180 秒**。也就是说，令牌从发出来到本地判定它过期，中间只隔了几十毫秒，本地却认为已经过去了四分钟。

第二类更直白：网关要求请求头带 `X-Timestamp` 做防重放，容忍窗口 60 秒。日志直接把 `skew=247s` 打出来了——本地时钟比网关快了 247 秒。

两条报错指向同一个方向：**时间**。但我们的第一反应是证书，因为"鉴权失败"最容易让人联想到 TLS。于是在容器里用 openssl 连了一次对方网关：

```bash
$ openssl s_client -connect api.gateway.example.com:443 \
    -servername api.gateway.example.com </dev/null 2>/dev/null \
    | grep -iE "verify return|notBefore|notAfter"
        notBefore=Aug  9 02:14:03 2026 GMT
        notAfter=Aug  9 02:14:03 2027 GMT
    Verify return code: 0 (ok)
```

证书链完整、校验通过、有效期到明年 8 月。证书这条线当场排掉。这一步很关键——**如果一开始就把"证书过期"这个假设坐实，后面会浪费大量时间在根本不存在的问题上**。顺便说，即便本地时钟快四分钟，也绝不可能让一张有效期一年的证书报过期，过期判断是 `now > notAfter`，差四分钟离到期还有一整年。

## 排查思路

证书排除之后，`skew=247s` 这行日志就成了唯一线索。容器默认共享宿主机的内核时钟，正常情况下两边的 `date` 应该只差毫秒级。我们在宿主机和容器里各打一次：

```bash
$ date '+%F %T %z'
2026-08-13 14:21:07 +0800
$ docker exec pay-adapter date '+%F %T %z'
2026-08-13 14:25:14 +0800
```

容器比宿主机快了四分零七秒，正好是 247 秒。而宿主机这边是干净的：

```bash
$ timedatectl status | grep -iE "synchronized|NTP service|Time zone"
               Time zone: Asia/Shanghai (CST, +0800)
     System clock synchronized: yes
               NTP service: active
$ chronyc tracking | grep -iE "System time|Last offset|Leap status"
System time    : 0.000021945 seconds slow of NTP time
Last offset    : +0.000011203 seconds
Leap status    : Normal
```

宿主跟 NTP 只差几十微秒，非常健康。

这就矛盾了：容器共享宿主内核时钟，宿主时间是准的，容器凭什么快四分钟？

只有一个解释——容器里读时间的那个调用根本没走到内核，被用户态的代码改写了。Linux 上最常见的实现就是 `libfaketime`：它通过 `LD_PRELOAD` 注入，拦截 `clock_gettime` 等库函数，对返回值加上一个偏移量。我们去看容器的环境变量：

```bash
$ docker exec pay-adapter env | grep -iE "faketime|LD_PRELOAD"
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/faketime/libfaketime.so.1
FAKETIME=+247
```

答案就在这里。`FAKETIME=+247` 的意思是：所有走 C 库时间函数的进程，看到的时间都加 247 秒。

## 处理过程

处理分三步。

**止血**：把部署配置里 `LD_PRELOAD` 和 `FAKETIME` 两个环境变量删掉，用干净的基础镜像重新构建并滚动重启。

```yaml
# deploy/pay-adapter.yaml —— 删除前
env:
  - name: LD_PRELOAD
    value: /usr/lib/x86_64-linux-gnu/faketime/libfaketime.so.1
  - name: FAKETIME
    value: "+247"
```

```bash
$ docker exec pay-adapter date '+%F %T %z'
2026-08-13 14:26:41 +0800
$ docker exec pay-adapter env | grep -c FAKETIME
0
```

两个 `date` 之间的差回到毫秒级，鉴权成功率当天回到正常水平。

**修根**：查这组变量是怎么进到生产里的。翻 Dockerfile 的历史提交，发现它们原本是

```dockerfile
# base-image/Dockerfile（历史版本）
ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/faketime/libfaketime.so.1
ENV FAKETIME=+0
```

写在测试用的基础镜像里，给历史回放测试伪造时间用的。后来这个基础镜像被复用到生产构建，`FAKETIME` 在生产的环境变量文件里被写成了 `+247`（大概是当初为了复现某个跨天的边界问题调的），一路带到了线上。因为 `ENV` 写进镜像配置，`docker exec` 进去也带着，`env` 一查就现形。

**加护栏**：
1. 基础镜像分层：生产镜像不再继承测试镜像，测试专用工具（faketime、各种 mock 库）通过多阶段构建隔离，不进最终镜像。
2. CI 里加扫描规则：生产镜像的 `ENV` 里出现 `LD_PRELOAD`、`FAKETIME`、`TZ` 之外的时钟相关变量，直接让流水线失败。
3. 监控加容器与宿主机的时间偏差指标，超过 5 秒告警。
4. 和网关侧沟通，把令牌校验的时钟容忍窗口从 60 秒放宽到 120 秒，给偶发抖动留余量。

## 根因分析

根因是两件事叠加：一处环境残留，加一处对时间的零容忍假设。

环境残留这一层比较好理解，但它的**隐蔽性**值得单独说。容器技术让所有进程共享宿主内核时钟，这件事太"理所当然"，以至于"容器里的时间一定是准的"成了一个没人验证的前提。而 `libfaketime` 这种用户态注入恰好绕开了这个前提——它不改内核，只在进程自己的地址空间里改写库函数返回值。你在宿主机上看时间、在容器里 `date`，两边看起来都有理，只有 `env` 能揭穿。

第二层是应用对时间毫无容错。支付网关的令牌有效期只有 180 秒，防重放窗口 60 秒，这是合理的安全设计；但我们的客户端假设"我和对端的时间完全一致"，这个假设在任何分布式系统里都不成立。正常情况下 NTP 把偏差压在毫秒级，假设看起来成立；一旦有 247 秒的偏移，整条链路立刻垮掉。

还有一个值得记下来的判断失误：我们一开始把"鉴权失败"直接映射成"证书问题"。这个联想在 90% 的场景下是对的，但它让排查绕了一圈。真正的线索 `skew=247s` 从第一条日志就在那儿摆着，是网关自己算出来告诉我们的——**对端给出的量化偏差，比我们自己的猜测值钱得多**。

## 复盘总结

事后我们在几个地方打了补丁。

第一，"容器时间与宿主机偏差"做成监控指标，超过 5 秒告警。这条成本极低，但能覆盖一整类问题——不只是 faketime，还包括宿主 NTP 失步、容器内误挂 `/etc/localtime`、时区搞错。

第二，基础镜像严格分层，测试专用工具不进生产；CI 加镜像扫描，`LD_PRELOAD` / `FAKETIME` 这类注入直接拦截。

第三，凡是接入外部鉴权的场景，客户端一侧统一加时钟容忍（我们定的是 120 秒），并在文档里写清楚"不要假设对端时钟与自己完全一致"。这是防御性编程里最容易被忽略的一条。

第四，runbook 里加了一句：看到"令牌过期""签名时间戳超限"，先打一次 `date`，再查 `env | grep -i faketime`。两秒钟的事，能省掉半天。

最有意思的一点是，这次故障几乎是"自己暴露"的——247 秒的偏移在容器里存在了很久，只是从来没遇到一个需要它的场景。事后回看，真正危险的不是那四分钟偏差，而是我们长时间以来对容器里那口钟的默认信任。系统里最容易被忽略的，往往正是那些"从来没出过问题"的假设。
