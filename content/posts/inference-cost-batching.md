---
title: "推理服务的成本账本：Prefill/Decode 分离与 Continuous Batching"
slug: "inference-cost-batching"
date: 2026-08-25
draft: false
description: "Token 经济爆发之后，推理成本成了硬指标。本文从 prefill 与 decode 两种截然不同的瓶颈出发，讲透 Continuous Batching 与 PD 分离为何能提升吞吐，并给出成本核算与指标监控的可运行示例。"
tags: ["推理优化", "Continuous Batching", "成本", "吞吐"]
categories: ["后端与架构"]
---

## 热点引入：175 万亿 Token/日背后的成本账

2026 年 8 月公开数据显示，国内日均 Token 调用量已达约 175 万亿。这个量级意味着，推理成本已经从"技术细节"上升为"商业模式能否成立"的核心变量——每百万 token 的单价下降一点点，乘以这个体量都是巨大的差额。

于是问题变得非常工程化：**同样的 GPU，怎样跑出更多 token？** 答案不在"换更快的卡"这么简单，而在于理解 LLM 推理的两个阶段有着完全不同的瓶颈，并据此设计调度方式。Continuous Batching 与 Prefill/Decode 分离（PD 分离）正是近年最重要的两项吞吐优化。本文把它们讲透，并给出成本与指标的度量方法。

## 技术原理：Prefill 与 Decode，两种截然不同的瓶颈

一次 LLM 推理分两步：

**Prefill（预填充）**：一次性并行处理整个输入 prompt，为所有输入 token 计算出第一份 KV Cache 和第一个输出 token。它的一次矩阵运算规模很大，属于**计算密集型（compute-bound）**，GPU 的算力被充分吃满。

**Decode（解码）**：随后逐个生成输出 token。每生成一个 token，都要读取此前所有的 KV Cache，但计算量极小——它属于**访存密集型（memory-bound）**，瓶颈在显存带宽而非算力。GPU 的算术单元大量闲置，利用率往往很低。

这个差异是一切优化的根源。把两个阶段混在同一批里调度，会让计算密集的 prefill 和访存密集的 decode 互相拖累：decode 等待 prefill 的长计算，prefill 又被 decode 的小步频打断。

由此引出四个关键指标：

- **TTFT（Time To First Token）**：从请求发出到第一个 token 返回，主要由 prefill 决定，影响"体感快慢"。
- **TPOT（Time Per Output Token）**：decode 阶段平均每 token 耗时，影响"输出流畅度"。
- **吞吐（tokens/s）**：单位时间产出的 token 总数，直接决定成本。
- **GPU 利用率**：算力和显存带宽的实际占用，反映资源浪费程度。

## 静态批处理与 Continuous Batching

传统静态批处理（static batching）把一批请求"绑在一起"：整批同时进入、整批全部生成完毕才释放。问题在于，不同请求的输入长度、输出长度差异极大。短请求早就生成完了，却必须陪着长请求占着显存和批次槽位——这就是"木桶效应"，GPU 大量空转。

**Continuous Batching（连续批处理 / 动态批处理）**打破了这种绑定：它按**迭代（iteration）**调度，每个 decode 步之后，已完成序列立即退出批次，空出的槽位马上填进新请求。批次组成在每一步都动态变化，因此得名。收益是决定性的：

- GPU 几乎不空转，吞吐大幅提升（常见数倍级提升，视负载而定）。
- 显存利用率更高，因为不再为已完成的序列保留 KV。
- 长短请求混跑时优势尤其明显。

它的代价是调度更复杂，且需要 PagedAttention 这类机制来管理离散、动态变化的 KV 内存块。这也是 vLLM 等引擎的核心能力。

## Prefill/Decode 分离（PD 分离）

既然 prefill 计算密集、decode 访存密集，最优解是把它们**部署到不同的 GPU 池**：

- Prefill 节点配置高算力卡，专门批量处理输入、产出 KV Cache。
- Decode 节点配置高带宽/大显存卡，专注逐 token 生成。
- 两者之间只传递 KV Cache（而非重复计算）。

收益有两层：其一，各自按瓶颈选型、独立扩缩容，资源不再错配；其二，避免长 prompt 的 prefill 阻塞正在 decode 的请求，显著改善 TPOT 的稳定性。代价是引入了 KV 跨节点传输的开销与系统复杂度——只有当 prefill 流量占比高、集群规模足够大时，收益才盖过成本。

## 实操一：vLLM 开启连续批处理

vLLM 默认启用连续批处理，核心是配好并发与批大小相关参数。

```bash
python -m vllm.entrypoints.openai.api_server \
  --model /models/my-llm \
  --served-model-name my-llm \
  --tensor-parallel-size 2 \
  --max-num-seqs 256 \
  --max-num-batched-tokens 8192 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.90 \
  --enable-prefix-caching \
  --port 8000
```

- `--max-num-seqs`：批次中同时存在的最大序列数，决定并发上限，越大越吃 KV 显存。
- `--max-num-batched-tokens`：单个迭代批次处理的最大 token 数，控制 prefill 的突发规模，影响 TTFT 与显存峰值。
- 两者要配合调整：并发高但 `max-num-batched-tokens` 太小，会导致 prefill 排队拉高 TTFT；反之则单次迭代太重，拖慢 decode。

## 实操二：吞吐与成本核算

把吞吐换算成钱，才能评估优化是否值得。下面按 token 单价与实测吞吐，估算每百万 token 成本。

```python
def cost_per_million_tokens(throughput_tps, gpu_count, gpu_hourly_price):
    """
    throughput_tps: 集群总吞吐（tokens/s）
    gpu_count: GPU 数量
    gpu_hourly_price: 单卡每小时价格（元）
    返回每百万 token 的算力成本（元）
    """
    tokens_per_hour = throughput_tps * 3600
    cluster_cost_per_hour = gpu_count * gpu_hourly_price
    return cluster_cost_per_hour / tokens_per_hour * 1_000_000

# 示例：8 卡集群吞吐 4000 tokens/s，单卡 20 元/小时
c = cost_per_million_tokens(throughput_tps=4000,
                            gpu_count=8, gpu_hourly_price=20)
print(f"每百万 token 算力成本: {c:.2f} 元")

# 吞吐翻倍 -> 成本减半，这就是所有吞吐优化的经济价值
c2 = cost_per_million_tokens(throughput_tps=8000,
                             gpu_count=8, gpu_hourly_price=20)
print(f"吞吐翻倍后: {c2:.2f} 元")
```

这个公式揭示了一个朴素事实：**在硬件与单价固定时，成本与吞吐成反比**。所以吞吐优化不是"技术洁癖"，而是直接砍账单。

## 实操三：关键指标计算

从请求日志里聚合 TTFT、TPOT 与吞吐，建立可观测基线：

```python
import statistics

def summarize(requests):
    """requests: [{'ttft':秒, 'output_tokens':整数, 'decode_time':秒}]"""
    ttft = [r["ttft"] for r in requests]
    tpot = [r["decode_time"] / max(r["output_tokens"], 1) for r in requests]
    total_tokens = sum(r["output_tokens"] for r in requests)
    total_time = sum(r["ttft"] + r["decode_time"] for r in requests)
    return {
        "p50_ttft_ms": round(statistics.median(ttft) * 1000),
        "avg_tpot_ms": round(statistics.mean(tpot) * 1000),
        "throughput_tps": round(total_tokens / total_time, 1),
    }

print(summarize([
    {"ttft": 0.2, "output_tokens": 100, "decode_time": 2.0},
    {"ttft": 0.3, "output_tokens": 150, "decode_time": 2.8},
]))
```

## 延迟与吞吐的权衡

不存在"延迟最低且吞吐最高"的魔力配置，二者常常冲突，必须按场景取舍：

- **在线交互**（客服、助手）：TTFT 和 TPOT 是硬约束。限制 `max-num-seqs` 与 `max-num-batched-tokens`，宁可牺牲一些吞吐，也要保证响应跟手。
- **离线批处理**（数据清洗、批量总结）：吞吐优先。放宽并发与批大小，把 GPU 榨干，对单个请求的延迟不敏感。
- **混合负载**：用队列隔离或独立实例区分在线与离线流量，避免大批量任务抢占交互请求的槽位。

## 工程实践建议与踩坑

- **先测基线再调参**。记录当前的 TTFT、TPOT、吞吐与 GPU 利用率，任何优化都要用数据证明有效，而不是凭感觉。
- **吞吐要看 P50 也要看 P99**。平均值掩盖长尾；P99 延迟劣化往往是拥塞或 KV 不足的早期信号。
- **PD 分离别过早引入**。它适合大规模、高 prefill 占比的场景；小规模集群里，跨节点传 KV 的开销可能抵消收益。
- **prefix caching 与连续批处理叠加收益明显**。公共 system prompt 被缓存后，prefill 缩短，TTFT 直接下降。
- **警惕"伪高吞吐"**。若吞吐上升但 P99 延迟飙升、请求大量超时，那不是优化，而是系统在过载边缘滑动。

## 小结

推理成本的账本，本质上是**吞吐的账本**。理解 prefill 计算密集、decode 访存密集这一根本差异，就能看懂 Continuous Batching 为何能靠动态调度大幅减少 GPU 空转，也能理解 PD 分离为何通过资源错配的纠正进一步提升效率。落到工程上，用 TTFT、TPOT、吞吐与 GPU 利用率建立基线，用 `max-num-seqs` 和 `max-num-batched-tokens` 精细调参，再按场景在延迟与吞吐间做明确取舍——这才是把"Token 经济"从口号变成可计算、可优化的真账本。
