---
title: "KV Cache 到底占了多少显存：从 MQA/GQA 到 FP4 量化"
slug: "kv-cache-deep-dive"
date: 2026-09-08
draft: false
description: "长上下文为何这么吃显存？答案藏在 KV Cache 里。本文从零推导 KV Cache 显存公式，讲清 MHA→MQA→GQA 的演进、FP8/FP4 量化与跨层共享，并给出可复用的显存计算工具。"
tags: ["KV Cache", "GQA", "量化", "显存优化"]
categories: ["后端与架构"]
---

## 热点引入：百万 token 上下文是怎么塞进单节点的

2026 年下半年，多家厂商在宣传自家旗舰模型时，都不约而同地把"百万 token 上下文装进单节点"当成卖点（具体型号与倍数以各家技术报告为准，本文不引具体数值）。支撑它的是一整套 KV Cache 压缩手段：低秩的潜向量注意力——把 K/V 压进一个远小于 hidden 维度的潜空间，MLA 是这类做法的代表；更激进的 KV 量化；以及跨层复用。几条路叠起来，每 token 的 KV 开销能被压掉一个数量级。

"KV Cache"这个原本偏底层的概念也因此被推到聚光灯下。

为什么长上下文的瓶颈总是显存，而不是算力？为什么优化长上下文，目标几乎总是"压 KV Cache"？要回答这些，必须回到 Transformer 解码的物理过程：每生成一个 token，模型都要用到此前所有 token 的 Key 和 Value。为了避免重复计算，这些中间结果被缓存下来——这就是 KV Cache。它随序列长度线性增长，很快就会超过模型权重本身，成为显存的主要消耗者。

本文把 KV Cache 的显存账算清楚，并梳理从 GQA 到 FP4 量化这条压缩主线。

## 技术原理：为什么要缓存 K/V，以及公式推导

自注意力计算中，每个 token 的 Query 要和所有历史 token 的 Key 做点积，再对 Value 加权求和。解码时，历史 token 的 K、V 是**不变**的——如果每次生成新 token 都重算一遍全部历史 K/V，计算量会随序列长度呈平方级爆炸。

KV Cache 的做法是：历史 token 的 K、V 算一次就存起来，后续每步只计算新 token 的 K、V 并追加。这样把每步计算从 O(n²) 降到 O(n)，代价是显存占用 O(n)。本质上，这是**用显存换计算**的经典权衡。

KV Cache 的显存占用可以精确推导。设：

- `L`：层数
- `S`：序列长度（token 数）
- `H_kv`：KV 头数（注意不是注意力头数！）
- `D_h`：每个头的维度（head_dim = hidden_size / num_attention_heads）
- `B`：每个元素的字节数（BF16=2，FP8=1，FP4≈0.5）
- 系数 `2`：K 和 V 各一份

则单条序列的 KV Cache 显存为：

```
KV_Cache_Bytes = 2 × L × S × H_kv × D_h × B
```

代入一组典型数字：L=32，S=32768，H_kv=32（标准 MHA），D_h=128，BF16（B=2）：

```
2 × 32 × 32768 × 32 × 128 × 2 ≈ 17.2 GB
```

**单条 32K 序列就要约 16 GiB 的 KV Cache**——这就是长上下文吃显存的直观原因。并发 10 条这样的请求，光 KV 就是 160 GB，远超一张 80GB 卡。

## MHA → MQA → GQA：从结构上减少 KV 头数

既然公式里 `H_kv` 是乘性因子，减少 KV 头数就是最直接的省显存手段。注意力机制的演进正沿着这条路走：

- **MHA（Multi-Head Attention）**：每个注意力头都有独立的 K、V 头，`H_kv = 注意力头数`。KV Cache 最大，上例中 H_kv=32。
- **MQA（Multi-Query Attention）**：所有查询头共享**一组** K、V，`H_kv = 1`。KV Cache 骤降为 MHA 的 1/32，显存极省；但共享过度，可能损伤模型质量。
- **GQA（Grouped-Query Attention）**：折中方案，把查询头分组，每组共享一对 K、V。例如 32 个查询头分成 8 组，`H_kv = 8`，KV Cache 降为 MHA 的 1/4。它在几乎不损失质量的前提下大幅省显存，因此成为当代主流（Llama、Qwen 等广泛采用）。

从 MHA 到 GQA，本质是在"每头独立 K/V 的表达力"与"KV 显存成本"之间找平衡。今天一个新模型公布时，`num_key_value_heads` 已是判断其长上下文成本的关键参数。

## KV 量化与跨层共享

结构优化之外，还有两类压缩手段：

**KV 量化**：把 K、V 从 BF16 降到 FP8（字节数 2→1，显存减半），甚至 FP4（2→0.5，降到 1/4）。收益巨大，但并非免费——低精度会引入误差，且 K/V 对精度较敏感，量化不当会导致生成质量下降。工程上通常用 per-channel 或分组缩放来缓解，并配合少量校准数据。

**跨层 KV 共享（跨层注意力复用）**：不同层复用同一份 K/V，而不是每层各存一份。若 N 层共享一份，理论上可再省下约 N 倍——注意是理论上限，层与层的表示差异越大损失越明显，实际能共享的层数有限。

这里要澄清一个常见误解：DeepSeek 系模型的 KV 压缩主力是 **MLA（Multi-head Latent Attention）**，靠低秩潜向量把每 token 的 K/V 从 `2 × 头数 × head_dim` 压到一个小得多的维度，跨层共享只是辅助手段。本文下面按"量化 + 跨层共享"推演，是为了把每一层压缩的贡献拆开看清楚，不代表真实模型的完整实现。

## 实操一：KV Cache 显存计算工具

下面这个函数输入模型配置，直接给出 KV Cache 估算，部署前算一算能避免很多意外。

```python
def kv_cache_gib(seq_len, num_layers, num_kv_heads, head_dim,
                 batch_size=1, bytes_per_elem=2):
    """计算 KV Cache 显存（GiB）。
    bytes_per_elem: BF16=2, FP8=1, FP4=0.5
    """
    bytes_ = (2 * seq_len * num_layers * num_kv_heads
              * head_dim * bytes_per_elem * batch_size)
    return bytes_ / 1024**3

# 典型场景：32K 上下文，32 层，head_dim=128
print("MHA(32 KV头), BF16:",
      round(kv_cache_gib(32768, 32, 32, 128), 2), "GiB")
print("GQA(8 KV头), BF16: ",
      round(kv_cache_gib(32768, 32, 8, 128), 2), "GiB")
print("GQA(8 KV头), FP8:  ",
      round(kv_cache_gib(32768, 32, 8, 128, bytes_per_elem=1), 2), "GiB")
print("GQA(8 KV头), FP4:  ",
      round(kv_cache_gib(32768, 32, 8, 128, bytes_per_elem=0.5), 2), "GiB")
```

## 实操二：量化在整条链上的收益

把 GQA、FP8、FP4 与跨层共享一层层叠加，看每一层各自贡献多少：

```python
base = kv_cache_gib(32768, 32, 32, 128)   # MHA + BF16 基线
variants = {
    "GQA(8头)":        kv_cache_gib(32768, 32, 8, 128) / base,
    "GQA+FP8":         kv_cache_gib(32768, 32, 8, 128, bytes_per_elem=1) / base,
    "GQA+FP4":         kv_cache_gib(32768, 32, 8, 128, bytes_per_elem=0.5) / base,
    "GQA+FP4+跨层4共享": kv_cache_gib(32768, 32, 8, 128,
                                     bytes_per_elem=0.5) / base / 4,
}
for name, ratio in variants.items():
    print(f"{name:20s} 相对基线 {ratio:.3f}x  (压缩约 {1/ratio:.1f} 倍)")
```

输出：

```text
GQA(8头)             相对基线 0.250x  (压缩约 4.0 倍)
GQA+FP8              相对基线 0.125x  (压缩约 8.0 倍)
GQA+FP4              相对基线 0.062x  (压缩约 16.0 倍)
GQA+FP4+跨层4共享      相对基线 0.016x  (压缩约 64.0 倍)
```

这四行值得逐行看：GQA 减头数是 4 倍，FP8 在 GQA 之上再翻一倍到 8 倍，FP4 再接一倍到 16 倍，跨层 4 层共享再乘 4 得到 64 倍。**"8 倍"只对应 GQA+FP8 这一行**，把 8 倍归因给"FP4 + 跨层复用"是错的——那两项单独叠加就已经是 64 倍。压缩倍数这种东西，一定要落在"相对谁"上：相对 MHA+BF16 是 64 倍，相对 GQA+BF16 就只有 16 倍。

从结构到精度再到跨层复用，多种手段相乘，才能把 KV 压到长上下文可行的量级。

## 实操三：从配置反推最大并发

部署时更关心"给定显存，能同时跑多少条长请求"：

```python
def max_concurrent(gpu_mem_gib, weight_gib, per_seq_kv_gib, util=0.9):
    usable = gpu_mem_gib * util - weight_gib
    return max(0, int(usable // per_seq_kv_gib))

# 80GB 卡，权重 40GB，单条 32K 请求 GQA+FP8 的 KV 约 2GiB
print("最大并发:",
      max_concurrent(80, 40, kv_cache_gib(32768, 32, 8, 128,
                                           bytes_per_elem=1)))
```

这个数字直接指导 `max-model-len` 与 `max-num-seqs` 的设定。

## 工程实践建议与踩坑

- **`max-model-len` 不要盲目设大**。它决定单条请求能占多长的 KV，实际长请求一多，就会挤占同一块 KV 池、把并发压下来。按业务 P99 长度设定，而不是照着模型最大值填。
- **PagedAttention 是隐形功臣**。它把 KV Cache 切成固定大小的块按需分配，避免预分配造成的显存碎片和浪费，让连续批处理中的变长序列能高效共存。开箱即用的引擎里务必确认它已启用。
- **量化要先验质量再上线**。FP4/FP8 对不同模型、不同任务的影响差异很大，务必在目标任务上做质量回归，别只看显存数字。
- **prefix caching 与 KV 正交**。缓存公共前缀能避免重复 prefill，但它本身也占显存，要纳入整体预算。
- **监控 KV 占用率**。它接近上限时，往往先于 OOM 出现延迟抖动，是扩容或限流的可靠信号。

## 小结

KV Cache 的显存=一条 `2 × 层数 × 序列长度 × KV头数 × head_dim × 精度字节 × 并发数` 相乘而来的账。理解了这个公式，你就明白长上下文为何吃显存、GQA 为何成为主流、量化与跨层共享为何能带来数量级的压缩。工程上没有银弹：结构上用 GQA 减头数，精度上用 FP8/FP4 降字节，系统上用 PagedAttention 与合理的 `max-model-len`/并发配置管好预算——多管齐下，才让百万 token 长上下文从"理论上可行"变成"单节点跑得动"。
