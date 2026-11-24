---
title: "灰度发布变成全量事故：Helm values 环境差异踩坑"
slug: "helm-gray-release-config-incident"
date: 2026-10-14
draft: false
description: "10 月一次本该只放 5% 流量的灰度发布，因为 values 里一个对不上的键名，把有问题的新版本全量推了出去。从 helm get values 查到回滚的整个过程。"
tags: ["Helm", "灰度发布", "values", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

10 月中旬，我们准备上线一个重构过的结算服务。新版本动了不少核心逻辑，团队心里没底，所以决定走灰度：先放 5% 流量观察，确认没问题再逐步放量。这套灰度能力是 Chart 里新加的——通过 Istio 的 VirtualService 按权重把流量分到 stable 和 canary 两个 subset。

Chart 模板写得很小心：`canary.enabled` 为 false 时只渲染一条指向 stable 的路由；为 true 时才按 `canary.weight` 拆权重。默认值 `enabled: false`、`weight: 0`，看起来是最保守的设计。

发布由我操作。我按文档准备好灰度配置文件，执行 `helm upgrade`，盯着网关的流量曲线等那 5% 上来。等来的不是 5%，而是流量几乎在一瞬间全切到了新版本——新版本里有个空指针问题，请求打进来就报 500，错误率转眼飙到三成以上。

## 异常现象

最直观的异常来自网关的流量分布：canary subset 的请求占比不是 5%，而是接近 100%。应用侧错误日志开始刷屏，全是新版本特有的那个空指针堆栈。

所以灰度分流机制本身是工作的——流量确实被导到了 canary——只是比例错了。第一反应是配置没按预期生效，先看这个 release 实际生效的 values：

```bash
$ helm get values settle-api -n prod -a | sed -n '/^canary:/,/^[a-z]/p'
canary:
  enabled: true
  percent: 5
  weight: 100
```

`percent: 5` 和 `weight: 100` 同时存在。我写的是 `canary.percent: 5`，而 Chart 模板真正用来渲染 VirtualService 权重的是 `canary.weight`。`percent` 这个键模板根本不认识，被当成无关数据忽略；`weight` 则取了 100。

于是"5%"的意图变成了"100%"的现实。

## 排查思路

第一件事是止血，但回滚之前得先确认上游状态，否则可能滚到更糟的版本。看历史：

```bash
$ helm history settle-api -n prod | tail -n 4
REVISION  UPDATED                   STATUS      CHART              DESCRIPTION
36        Mon Aug 24 11:02:19 2026  superseded  settle-api-2.3.1   Upgrade complete
37        Tue Oct 14 14:05:47 2026  deployed    settle-api-2.4.0   Upgrade "gray 5%"
```

37 就是刚才这次灰度——它是 2.4.0，出问题的新版本；36 是八月上线、一直跑到今天的 2.3.1。要回到稳定态，得回滚到 **36**。

这里顺带说个容易看错的点：`helm history` 默认最多显示 256 条，用 `tail` 只是截了个尾。这个 release 从去年到现在已经滚了三十几次，别看到"36"就以为发过三十多次事故——它只是总第 36 次发布。

这里有个必须搞清楚的问题：`weight: 100` 是从哪来的？Chart 的默认值是 0，我写的灰度文件里也没有 `weight` 这个键。于是去看两个 values 文件的实际内容：

```yaml
# values-prod.yaml（生产基线，节选）
canary:
  enabled: false
  weight: 100      # ← 半年前灰度能力上线时留的占位值，从没被用过
```

```yaml
# values-gray.yaml（本次灰度，节选）
canary:
  enabled: true
  percent: 5       # ← 键名写错了，模板认的是 weight
```

真相清楚了。`weight: 100` 是半年前灰度能力刚上线时，有人在生产基线里留的占位值。因为 `enabled: false` 时模板压根不渲染权重拆分，这个 100 一直躺在配置里，从来没起过作用——直到这次 `enabled` 被翻成 true，它立刻生效。

团队里有人提出另一个合理怀疑：会不会是多个 `-f` 文件的覆盖顺序搞反了？我们的发布命令确实带了两个文件：

```bash
$ helm upgrade settle-api ./chart/settle-api -n prod \
    -f values-prod.yaml \
    -f values-gray.yaml \
    --reuse-values
```

按 Helm 规则，后面的 `-f` 覆盖前面的，灰度文件在后，顺序本身没问题。但 `--reuse-values` 这一项让事情变复杂了：它会把上一次 release 的全部 values 作为底稿，再让你用 `-f` 在上面覆盖。也就是说，`values-prod.yaml` 里的 `weight: 100` 不但被加载，还被当成"上次的状态"继承了下来。

三个因素凑齐了：占位值 100（隐患）+ 键名写错（直接原因）+ `--reuse-values`（让旧键顽固地留下来）。

## 处理过程

回滚：

```bash
$ helm rollback settle-api 36 -n prod
Rollback was a success! Happy Helming!
$ helm history settle-api -n prod | tail -n 3
REVISION  UPDATED                   STATUS      CHART              DESCRIPTION
37        Tue Oct 14 14:05:47 2026  superseded  settle-api-2.4.0   Upgrade "gray 5%"
38        Tue Oct 14 14:19:02 2026  deployed    settle-api-2.3.1   Rollback to 36
```

注意回滚并没有"回到 36"——它生成了一个新版本 38，内容等于 36。这条语义很关键：**Helm 的回滚是"用旧版本的配置再发一次"，不是把历史指针拨回去**。所以 `helm rollback settle-api 37` 会把那套坏配置再发一遍，越滚越糟。

流量在几十秒内切回稳定版本，错误率回落。

接着修正配置。三处一起改：灰度文件把 `percent` 改成 `weight`；生产基线里那个没用的 `weight: 100` 删掉，改成 0；发布命令去掉 `--reuse-values`，避免旧值被继承。

然后——这是这次最该补的环节——发布前先 diff 一遍渲染结果：

```bash
$ helm diff upgrade settle-api ./chart/settle-api -n prod \
    -f values-prod.yaml -f values-gray.yaml | grep -A8 -B2 "route"
  spec:
    http:
    - route:
      - destination:
          host: settle-api
          subset: stable
+       weight: 95
+     - destination:
+         host: settle-api
+         subset: canary
+       weight: 5
```

因为回滚后线上是 `enabled: false` 的单路由形态，diff 表现为"一条无权重路由"变成"stable 95 / canary 5 两条"，全部是新增行——这正是我们期望看到的形态。如果 diff 里出现 `- weight: 100` 这种**旧值**，反而说明基线里已经带了权重，也就是那个占位值又开始作祟了，属于必须停下来查的信号。确认无误后再执行 upgrade，这次 5% 稳稳地放上去，观察半小时无异常，才按计划逐步放量。

顺带给 Chart 加了 values schema 校验，未知键直接报错：

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "canary": {
      "type": "object",
      "properties": {
        "enabled": { "type": "boolean" },
        "weight":  { "type": "integer", "minimum": 0, "maximum": 100 }
      },
      "additionalProperties": false
    }
  }
}
```

`additionalProperties: false` 这一行，就是让 `percent` 这种笔误在 `helm lint` 阶段直接失败，而不是安静地在生产里跑一遍。

## 根因分析

把整件事拆开看，有四层。

最表层是一个键名写错了。`canary.percent` 与 Chart 认的 `canary.weight` 差了一个词。这种错在 YAML 里不会报任何错——Helm 默认不会因为多出一个未知键就拒绝渲染，它安安静静地被忽略。**静默失败是配置系统最危险的特性**：你以为写了，系统以为没写，谁都不觉得有问题。

第二层是那个占位值。`weight: 100` 藏在生产基线里半年，因为 `enabled: false` 时它不参与渲染，所以从来没被验证过。这是一个"看起来无害、一旦激活就是全量"的定时炸弹。更荒谬的是它的值：一个面向灰度场景的参数，占位值取的是最激进的 100，而不是最保守的 0。

第三层是 `--reuse-values`。它的语义是"以上一次 release 的 values 为底稿"，这意味着 Chart 新版本里的默认值不会生效，历史遗留的键反而会被完整继承。我们本意是"少写几个参数"，实际效果是"把所有历史包袱都带着走"。

第四层是流程——没有发布前 diff。这是唯一一道能在按下回车之前发现问题的闸门，我们跳过了。事后复盘，这一条比前面三条都更值得反省：前三条是知识和习惯问题，这一条纯粹是流程缺失。

## 复盘总结

事故收尾后，我们定了五条规矩。

第一，所有发布必须先 `helm diff upgrade` 核对渲染差异，diff 结果人工确认后才允许执行 upgrade。这条写进了发布流程的强制卡点，不 diff 不给发。

第二，**灰度类参数的默认值和占位值一律取最保守的那个**。`weight` 的默认必须是 0，`enabled` 必须是 false。什么时候放量，由当次发布显式写清楚，而不是靠某个地方藏着的一个 100。

第三，给 Chart 加 values schema，未知键直接报错。用 JSON Schema 的 `additionalProperties: false` 把"笔误"变成"构建失败"。

第四，禁用 `--reuse-values`。需要保留某项配置就显式写进 values 文件，让每次发布的状态都是完整、可读、可 review 的，而不是"上次那样再加一点"。

第五，配置集中管理，各环境的 values 统一放一个仓库走 MR 评审，禁止本地随手拼 `-f` 文件。

灰度发布的意义，本来就是把风险切成小块一点点试。这次我们因为一个键名，把整块风险一口吞了下去。所幸新版本的问题只是报错、没有造成数据损坏，回滚也够快。但教训是清楚的：在配置驱动的系统里，写错一个字和按下发布键之间，不该没有任何距离。
