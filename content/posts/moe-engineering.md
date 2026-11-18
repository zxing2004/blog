---
title: "万亿参数 MoE 的三个工程支点：专家并行、负载均衡与通信"
date: 2026-07-08
draft: false
description: "以 Kimi K3 全量开源为契机，讲透 MoE 稀疏激活的核心原理，并逐一拆解支撑万亿参数模型落地的三个工程支点：专家并行、负载均衡与通信优化。"
tags: ["MoE", "专家并行", "大模型", "分布式"]
categories: ["后端与架构"]
---

## 热点引入：Kimi K3 开源，把 MoE 再次推到台前

2026 年 7 月，月之暗面将 Kimi K3 全量开源，2.8 万亿参数使其成为全球最大开源模型。社区讨论焦点除了其接近闭源旗舰的定性能力，更多落在一个架构事实上：如此体量几乎不可能沿用稠密（dense）结构，MoE（混合专家）几乎是唯一可落地的工程答案。

这并非孤例：近两年的旗舰模型一旦跨过千亿参数门槛，几乎清一色转向 MoE。原因很朴素——稠密模型每生成一个 token 都要激活全部参数，参数量约等于计算量；而 MoE 让"模型有多大"与"算一次有多贵"解耦。理解它，是读懂当代大模型工程的前提。

本文不谈能力排行，而是聚焦决定 MoE 能否跑起来的三个工程支点：专家并行、负载均衡与通信优化。

## 技术原理：稀疏激活如何解耦参数量与计算量

MoE 的核心思想是**用多个"专家"子网络替换 Transformer 中的前馈网络（FFN）层，但每个 token 只激活其中少数几个专家**。

一个 MoE 层由两部分组成：

- **Router（门控网络）**：一个极小的线性层，对每个 token 的隐藏向量打分，输出它在各个专家上的权重，再取 top-k（常见 k=1、2、8）。
- **Experts（专家）**：一组结构相同、参数独立的 FFN。被选中的专家对 token 做变换，输出按 router 权重加权求和。

由此产生两个关键数字：

- **总参数量（total params）**：所有专家参数之和，决定模型"记忆容量"和显存占用。Kimi K3 的 2.8 万亿说的就是这个。
- **激活参数量（active params）**：单个 token 前向实际经过的参数，决定单步计算量（FLOPs）。以 top-2、共 64 个专家为例，一次只激活约 2/64 的专家参数。

稀疏激活有两个直接收益：**计算量与参数量解耦**——总参数可堆到万亿级提升容量，单 token 计算量却仍停留在百亿级稠密模型的水平；**训练效率更高**——同样算力预算下，MoE 能容纳远超稠密模型的参数量。

但稀疏不是免费的：它把"算得少"换成了"通信重、难均衡、显存放不下"的难题——这正是三个支点要解决的。

## 支点一：专家并行（Expert Parallelism）

当专家数量多到单卡放不下时，就要把专家切分到不同 GPU 上，这就是**专家并行（EP）**。它与张量并行（TP）、流水线并行（PP）并称三维并行，但通信模式截然不同。

EP 中，token 的路由是动态的：每个 token 可能去任意专家，而专家分布在各卡上。因此需要在 **dispatch（分发）** 阶段把 token 按目标专家重排并通过 `all-to-all` 通信送到对应卡，专家计算完后再用一次 `all-to-all` 把结果**聚合（combine）**回来。all-to-all 是 EP 的命脉，也是主要瓶颈：每张卡都要和其他所有卡交换数据，通信量随 `token 数 × hidden 维度 × EP 并行度` 增长。

工程上 token 在 dispatch 前要按专家 ID 排序、打包成定长块，才能高效 all-to-all；零散小消息会让带宽利用率骤降。这也是 DeepSpeed、Megatron-LM 等框架对 `all-to-all` 做大量优化的原因。

## 支点二：负载均衡（Load Balancing）

Router 天然存在"马太效应"：少数专家被反复选中，多数闲置。负载不均既浪费算力，又让少数卡成为木桶短板。业界主要有三种手段：

1. **辅助负载均衡损失（load balancing loss）**：在主损失外叠加一项惩罚，鼓励 token 在各专家间均匀分配。它是训练稳定的关键，但权重过大会损伤模型质量，需要精细调参。
2. **容量因子（capacity factor）**：给每个专家设定容量上限 `capacity = ceil(token数 / 专家数 × capacity_factor)`。超出容量的 token 会被丢弃（token dropping）。
3. **专家选择（expert choice）路由**：反转路由方向，让每个专家"挑选"固定数量的 token，从机制上保证严格均衡，代价是可能某些 token 被零个或过多专家处理。

推理阶段通常不用辅助损失，但要靠容量配置和合理的 router 温度来避免极端倾斜。

## 支点三：通信优化

all-to-all 的成本可以靠两类手段压低：

- **计算与通信重叠**：把 dispatch 的通信与上一层计算、把 combine 的通信与专家计算重叠起来，用流水掩盖延迟。
- **分层通信**：在单机内用 NVLink/NVSwitch 做高速交换，跨节点才走 InfiniBand/RoCE。让通信尽量发生在带宽更高的层级，避免跨节点 all-to-all 成为瓶颈。

## 实操：top-k Router 与 MoE FFN 的最小实现

下面用 PyTorch 实现一个简化版 MoE 层，完整包含 top-k 路由、专家计算与加权合并。为突出原理，这里不写 all-to-all（单机伪分布式），专家用 `ModuleList` 表示。

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class Expert(nn.Module):
    """单个 FFN 专家：升维 -> 激活 -> 降维"""
    def __init__(self, d_model, d_ff):
        super().__init__()
        self.up = nn.Linear(d_model, d_ff)
        self.down = nn.Linear(d_ff, d_model)

    def forward(self, x):
        return self.down(F.gelu(self.up(x)))

class MoELayer(nn.Module):
    def __init__(self, d_model, d_ff, num_experts, top_k=2):
        super().__init__()
        self.top_k = top_k
        self.gate = nn.Linear(d_model, num_experts)          # router
        self.experts = nn.ModuleList(
            [Expert(d_model, d_ff) for _ in range(num_experts)]
        )

    def forward(self, x):
        # x: [num_tokens, d_model]
        logits = self.gate(x)                                 # [T, E]
        topk_val, topk_idx = torch.topk(logits, self.top_k, dim=-1)
        weights = F.softmax(topk_val, dim=-1)                 # 仅对选中的专家归一化

        out = torch.zeros_like(x)
        for k in range(self.top_k):
            expert_id = topk_idx[:, k]
            w = weights[:, k].unsqueeze(-1)
            for e in range(len(self.experts)):
                mask = (expert_id == e)
                if mask.any():
                    tokens = x[mask]
                    out[mask] += w[mask] * self.experts[e](tokens)
        return out

# 冒烟测试
layer = MoELayer(d_model=512, d_ff=2048, num_experts=8, top_k=2)
x = torch.randn(16, 512)
y = layer(x)
print(y.shape)   # torch.Size([16, 512])
```

真实系统会把逐专家循环替换成"排序 + 分组 GEMM + all-to-all"融合算子，否则循环本身就是瓶颈。

### 专家容量与 token 丢弃（伪代码）

```text
function route_with_capacity(logits, k, capacity_factor):
    E = number_of_experts
    T = number_of_tokens
    topk_idx = topk(logits, k)                    # [T, k]

    # 1) 统计每个专家被分配到的 token 数
    capacity = ceil(T / E * capacity_factor)
    count = zeros(E)

    keep_mask = full([T, k], False)
    for token in range(T):
        for slot in range(k):
            e = topk_idx[token][slot]
            if count[e] < capacity:               # 还有余量就接收
                keep_mask[token][slot] = True
                count[e] += 1
            # 否则该 token 的这一路被丢弃 (token dropping)

    # 2) 被丢弃的 token 走残差直连（或降级到次优专家）
    #    out = residual + MoE(x)  中 MoE 分支对丢弃项贡献 0
    return topk_idx, keep_mask, count
```

被丢弃的 token 不会消失，而是以残差直连穿过该层——代价是没获得专家处理能力，这也是容量因子不能太小（常用 1.0～1.25）的原因。

## 显存与通信开销的定量分析

**权重显存**：以 BF16（每参数 2 字节）为例，2.8 万亿参数仅权重就需要 `2.8e12 × 2 ≈ 5.6 TB`，必须靠数十张 80GB 级 GPU 分片承载。若量化到 FP8（1 字节）则约 2.8 TB，量化的收益是数量级的。

**通信量**：一次 EP all-to-all 的单卡收发量近似 `T × d_model × bytes × (EP-1)/EP`。前向 dispatch 与 combine 各一次，反向再各一次，单步通信量因此成倍放大；EP 从 8 增到 64，跨节点流量也随之猛增。下面这个函数把权重与通信量算在一起：

```python
def moe_cost(params_b, ep, tokens_per_gpu, hidden, bytes_per_param=2):
    """估算权重显存(TB)与单步 all-to-all 通信量(MB)"""
    weight_tb = params_b * 1e9 * bytes_per_param / 1e12
    comm_mb = tokens_per_gpu * hidden * bytes_per_param * (ep - 1) / ep / 1e6
    return round(weight_tb, 2), round(4 * comm_mb, 1)  # 前反向共 4 次

print(moe_cost(2800, ep=8, tokens_per_gpu=4096, hidden=4096))
```

**激活与 KV Cache**：激活显存随并发和序列长度线性增长，推理时 KV Cache 往往比权重更棘手，是长上下文场景的头号显存挑战。

## 工程实践建议与踩坑

- **EP 度不是越大越好**。EP 越大单卡放的专家越省、可选专家越丰富，但 all-to-all 开销快速上升，要用实测找到"收益 > 代价"的拐点。
- **优先保证通信与计算重叠**。检查框架的 overlap 开关是否打开；没打开时 all-to-all 是串行阻塞的，GPU 大量空转。
- **aux loss 权重需谨慎调整**。太小不均衡，太大伤质量；训练中建议监控各专家的 token 占比直方图，出现"专家饥饿"要及时干预。
- **容量因子留余量**。推理 peak 流量下 token 分布更极端，capacity_factor 过小会大量丢 token，表现为质量突然下降。
- **top-k 的选择**。k=1 通信最省但质量与均衡性较差；k=2 是常见甜点；更大 k 提升质量但通信与计算翻倍。
- **专家中间结果别在 Python 层循环**。生产环境务必用框架提供的融合 MoE kernel。

## 小结

MoE 用稀疏激活把"参数量"与"计算量"解耦，这是万亿参数模型可行的根本原因。但稀疏不是免费的：让专家分布起来要靠**专家并行**，让每个专家充分使用要靠**负载均衡**，让 all-to-all 不拖后腿要靠**通信优化**。三者缺一不可——它们决定了 Kimi K3 这类模型，究竟是纸面参数还是能跑出吞吐与性价比的工程系统。理解它们，才算真正理解当代大模型的基础设施。
