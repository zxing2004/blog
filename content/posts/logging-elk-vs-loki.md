---
title: "日志系统选型：ELK 与 Loki 的取舍"
slug: "logging-elk-vs-loki"
date: 2026-06-19
draft: false
description: "对比 ELK 与 Loki 在采集链路、查询能力、存储成本上的差异，给出选型建议与 Filebeat、LogQL、Painless 实操示例。"
tags: ["日志", "ELK", "Loki", "可观测性"]
categories: ["运维部署"]
---

## 日志系统选型的两条路线

"我们要上日志平台"这句话一出，团队往往立刻分成两派：一派说用 ELK，功能强大、搜索快；另一派说用 Loki，轻量省钱、和 Grafana 原生集成。两边都能列出一堆理由，最后往往变成信仰之争。

其实两者代表的是两种设计哲学。ELK（Elasticsearch + Logstash + Kibana）为每个字段建索引，换来全文检索与复杂聚合的自由，代价是存储与计算开销大。Loki 只索引标签（label），日志内容原样压缩存储，用"先按标签缩小范围、再在内容里做线性过滤"的思路，把成本压到很低。理解这个根本差异，选型就变成了结合自己需求的计算题，而不是站队。

## 采集链路：从 Agent 到存储

两条路线的采集层其实高度相似，通常用 Filebeat 或 Fluent Bit 作为边车（sidecar）或节点级 Agent：

```yaml
# Filebeat 采集容器日志并打上 Kubernetes 元数据
filebeat.inputs:
  - type: filestream
    id: container-logs
    paths:
      - /var/log/containers/*.log
    parsers:
      - container: ~
    processors:
      - add_kubernetes_metadata:
          matchers:
            - logs_path:
                logs_path: "/var/log/containers/"

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "logs-%{+yyyy.MM.dd}"

# 输出到 Loki 时改用 loki output，并配置 labels
output.loki:
  hosts: ["http://loki:3100"]
  labels:
    namespace: "${kubernetes.namespace}"
    pod: "${kubernetes.pod.name}"
    container: "${kubernetes.container.name}"
```

采集层的关键决策是"结构化在哪做"：ELK 生态常用 Logstash/Ingest Node 在写入前解析 JSON、抽取字段；Loki 则鼓励在 Agent 侧用 `processors` 打标签，内容保持原样。

## 查询能力对比

ELK 的强项是全文检索与结构化聚合。用 KQL/Lucene 查询、Kibana Dashboard 做可视化，复杂分析可用 Painless 脚本：

```json
// Elasticsearch 中检索 error 级别并按 service 聚合
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [{ "match": { "level": "ERROR" } }],
      "filter": [{ "range": { "@timestamp": { "gte": "now-1h" } } }]
    }
  },
  "aggs": {
    "by_service": { "terms": { "field": "service.keyword", "size": 10 } }
  }
}
```

Painless 脚本可在查询时做字段加工，例如把耗时按阈值分档：

```java
// Painless scripted field：把 duration_ms 映射为快慢档
if (doc['duration_ms'].value > 1000) return 'slow';
return 'fast';
```

Loki 用 LogQL，思路是先标签过滤、再内容匹配：

```logql
# 标签选择 + 内容行过滤（|= 包含，!= 不包含，|~ 正则）
{namespace="prod", app="payment"} |= "timeout" != "healthcheck"

# 统计日志速率并按 level 分组
sum by (level) (rate({app="payment"} | json | line_format "{{.level}}" [5m]))

# 提取 JSON 字段后过滤 http_status >= 500
{app="gateway"} | json | status >= 500
```

LogQL 的 `{label="value"}` 是索引部分，决定扫描范围；`|=` 之后是线性过滤——所以标签设计得好不好，直接决定查询快慢。

## 存储成本与保留策略

ELK 为每个字段建倒排索引，索引本身常与原始日志相当甚至更大，加上副本，存储成本是 Loki 的数倍。它适合对检索灵活性要求高、日志量中等、能接受较高成本的场景。Loki 只对标签建索引、日志体用类似 Prometheus 的块存储压缩，对象存储（S3 等）即可承载，成本极低，适合日志量大、查询模式固定（按服务/命名空间查）的场景。

保留策略上，ELK 常用 ILM（索引生命周期管理）做热温冷分层，Loki 则通过 `retention_period` 配合对象存储生命周期规则实现。

## 何时选谁

选 ELK：需要全文检索、复杂聚合分析、日志字段频繁变化、团队有 Elasticsearch 运维能力。
选 Loki：日志以排查为主、查询围绕服务/标签、成本敏感、已用 Grafana 全家桶、希望和指标共享一套告警。

两者也常共存：ELK 承载需要深度分析的安全/审计日志，Loki 承接海量应用访问日志。

## 采集可靠性与脱敏

采集链路的可靠性常被忽视。Agent 应有本地磁盘缓冲，在后端短暂不可用时先落盘、恢复后补发，避免日志丢失；但同时要设置队列上限与丢弃策略，防止缓冲无限增长拖垮节点。写入 Elasticsearch 前最好经过 ingest pipeline 做字段归一化与敏感信息脱敏，把手机号、身份证等字段掩码后再入库。还要为日志量设置基于服务的配额与告警：某个服务突然把日志量放大十倍，往往是 bug 或异常循环的信号。把日志系统本身也当成最灵敏的业务探针之一，它才能在救火时真正派上用场。

## 踩坑与最佳实践

Loki 的标签基数必须克制——把 user_id、request_id 当标签会导致索引爆炸，高基数字段应写进日志内容用 `| json` 过滤。

ELK 的 `keyword` 与 `text` 字段要分清：需要精确聚合用 `keyword`，需要分词全文检索用 `text`，混用是查询慢的常见原因。

无论哪种方案，采集端都要设置丢弃策略与限流，避免某服务疯狂打日志把整条链路冲垮。

日志时间戳统一为 UTC 并保证客户端时钟同步（NTP），否则跨服务关联会得出错误结论。

## 小结

ELK 与 Loki 不是谁替代谁，而是"自由度"与"成本"的权衡：ELK 用索引换来任意字段的全文检索与聚合，Loki 用标签索引换来极低的存储成本和与 Grafana 的无缝集成。先明确自己究竟是"要做日志分析"还是"要能快速查错"，再对照查询模式与预算做选择。把标签设计好、把保留策略定好，日志系统才能从"存了一堆没人看的数据"变成真正能救火的排障利器。
