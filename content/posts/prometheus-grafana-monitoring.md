---
title: "用 Prometheus + Grafana 搭建监控告警体系"
slug: "prometheus-grafana-monitoring"
date: 2026-04-16
draft: false
description: "从四类指标与 PromQL 到 exporter 抓取、Alertmanager 路由抑制与 Grafana 面板，搭建一套可用的监控告警体系。"
tags: ["Prometheus", "Grafana", "监控告警", "PromQL"]
categories: ["运维部署"]
---

## 监控不是把图表堆满屏幕

很多团队的监控现状是：Grafana 上挂了几百张图，平时没人看，出事时也没人能从图里看出根因。指标采了一堆，告警却要么从不响，要么半夜响成一片。问题不在工具，而在没有体系：不知道该采集什么、怎么查询、什么时候告警、告警发给谁。

一套有效的监控要回答三个问题：系统现在是否正常（仪表盘）、哪里不正常（查询与下钻）、要不要叫人以及叫谁（告警）。Prometheus 负责采集与存储时序数据，Alertmanager 负责告警的聚合、路由与抑制，Grafana 负责可视化。三者各司其职，拼起来才是完整闭环。

## 四类指标与 PromQL 基础

Prometheus 的指标有四种类型，选对类型是写对查询的前提：

- **Counter（计数器）**：只增不减，如请求总数、错误数。用 `rate()`/`increase()` 算速率。
- **Gauge（仪表盘）**：可增可减，如当前温度、队列长度、内存使用。
- **Histogram（直方图）**：把观测值分桶累计，用于计算分位数与分布，如请求耗时。
- **Summary（摘要）**：客户端直接计算分位数，开销高且不可聚合，场景有限。

常用 PromQL：

```promql
# 每秒 HTTP 请求速率，按 handler 聚合
sum(rate(http_requests_total[5m])) by (handler)

# 5 分钟错误率
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# 按 service 统计 95 分位响应延迟（直方图）
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# CPU 使用率百分比
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

`rate()` 会自动处理计数器重置（进程重启归零），所以算 Counter 速率一定要用它而不是直接相减。`histogram_quantile` 依赖 `_bucket` 时间序列的 `le` 标签做插值，注意桶边界设置要覆盖真实延迟范围。

## 抓取配置与 exporter

Prometheus 采用拉（pull）模型，主动去目标端点拉取 `/metrics`。在 `prometheus.yml` 中配置：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["10.0.0.11:9100", "10.0.0.12:9100"]

  - job_name: "business_app"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["10.0.1.20:8080"]
```

主机指标由 `node_exporter` 暴露，应用指标则通过客户端库（如 Java 的 Micrometer、Go 的 prometheus/client_golang）埋点。服务动态扩缩容场景应改用服务发现（`kubernetes_sd_configs`、`consul_sd_configs`），避免写死 target。

## 告警规则与 Alertmanager 路由

告警规则同样用 YAML 表达，带 `for` 字段可抑制抖动：

```yaml
groups:
  - name: service-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.job }} 5xx 错误率超过 5%"
          runbook: "https://wiki.example.com/runbook/high-error-rate"

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} 无法抓取"
```

Alertmanager 负责把告警按标签路由到不同接收渠道，并做去重与抑制：

```yaml
route:
  group_by: ["alertname", "job"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "default"
  routes:
    - matchers: ["severity=critical"]
      receiver: "oncall-pager"

inhibit_rules:
  - source_matchers: ["alertname=InstanceDown"]
    target_matchers: ["severity=warning"]
    equal: ["instance"]

receivers:
  - name: "oncall-pager"
    webhook_configs:
      - url: "https://alert.example.com/pager"
  - name: "default"
    email_configs:
      - to: "ops@example.com"
```

`inhibit_rules` 让"实例已宕机"这类根因告警抑制掉同一实例上的一切衍生告警，避免告警风暴。Grafana 则通过添加 Prometheus 数据源，用变量与面板把上面的查询可视化，并与 Alertmanager 联动展示告警状态。

## 监控自身的健康

抓取链路自身的健康是监控的"监控"。Prometheus 的 `up` 指标表示每次抓取是否成功，应把 `up == 0` 列为最基础的告警——否则 exporter 挂了你却浑然不知，监控盲区比没有监控更危险。服务发现场景下实例地址是动态生成的，需要用 `relabel_configs` 把元标签改写为统一的业务标签（如 env、team），同时过滤掉不想采集的目标。抓取间隔也要权衡：15 秒对多数故障排查足够，过短会显著放大抓取与存储压力。此外，Prometheus 本地存储对磁盘较敏感，规整的磁盘规划（SSD、独立分区、预留空间）能避免数据目录写满导致的数据丢失。

## 踩坑与最佳实践

Counter 名称以 `_total` 结尾是规范也是许多工具箱的约定，直方图必须暴露 `_bucket`、`_sum`、`_count` 三类序列，缺一不可。

`for` 太短会让网络抖动触发误报，太长会延迟真实故障，建议从 2-3 分钟起步再按历史误报率调整。

高基数（label 取值过多，如把 user_id 当标签）会让时序数量爆炸，直接压垮 Prometheus。标签只放低基数维度，明细数据交给日志系统。

Prometheus 本地存储适合短期数据（默认 15 天），长期趋势应接 Thanos 或 Mimir，别把它当归档库用。

## 小结

监控告警体系的搭建顺序是：先用四类指标规范埋点，再用 PromQL 把"正常"定义成查询，接着用规则文件把阈值固化成告警，最后交给 Alertmanager 路由抑制、Grafana 呈现。记住监控的目标不是收集最多的数据，而是在故障发生时最短路径地回答"哪里坏了、严不严重、该谁处理"。围绕这个目标设计，你的仪表盘才不会沦为墙上的装饰。
