---
title: "原生多模态的工程实现：从编码器对齐到统一 token 空间"
date: 2026-09-19
draft: false
description: "原生多模态已成旗舰标配，但它与『图像编码器 + 投影层』的拼接方案究竟差在哪？本文拆解 ViT patch 化、模态对齐、视频时序编码等关键实现，并给出可运行的代码骨架与自研/API 决策建议。"
tags: ["多模态", "视觉语言模型", "对齐", "架构"]
categories: ["后端与架构"]
---

## 热点引入：原生多模态成为旗舰标配

2026 年下半年，几家头部厂商的新一代旗舰模型密集发布，文本、图像、视频的统一理解能力不再是可选项，而是旗舰的标配。"原生多模态"这个词随之频繁出现在发布材料里（具体型号与版本号以各厂商官方发布为准，本文不点名）。

但对工程团队来说，"原生"二字需要拆开看：它到底是一种架构差异，还是仅仅营销话术？真正的区别，在于模型是**先把图像硬塞进文本空间再对齐**，还是**从训练一开始就让多种模态在同一个 token 空间里联合建模**。这个差异决定了能力上限、训练成本，以及你该调用 API 还是自己造轮子。

本文从工程实现角度，讲清多模态模型是怎么把图像、视频变成 LLM 能处理的 token 的，并给出从 ViT 切 patch 到统一 token 空间的代码骨架。

## 技术原理：拼接式与原生多模态

**拼接式多模态（modular / adapter-based）**是早期主流方案：用一个预训练好的视觉编码器（如 ViT）提取图像特征，再接一个投影层（projector）把特征映射到 LLM 的词嵌入空间，然后和文本 embedding 拼在一起喂给 LLM。它的优点是**复用现成的强视觉编码器和强 LLM，训练成本相对低**；缺点是两者是"婚后磨合"——对齐层只能部分弥合两个独立预训练空间的鸿沟，细粒度理解和复杂推理存在天花板。

**原生多模态（native multimodal）**则从训练之初就把多种模态统一成 token 序列，在同一套 Transformer 里联合训练。图像、文本、视频对自己模态的 token 建模，也学习跨模态的关联。它避免了"拼接缝隙"，在跨模态推理、细粒度理解上潜力更大；代价是训练数据与算力需求极高，工程复杂度陡增。

取舍很清楚：追求快速落地和成本可控，拼接式足够；追求能力上限且有充足资源，才谈原生。

## 视觉编码：ViT 如何把图像切成 token

LLM 只认 token 序列，图像必须先被"离散化"成 token。Vision Transformer（ViT）的做法是 **patch 切分**：把一张 H×W 的图像切成 P×P 的小块，每块展平后经线性投影成一个向量，就得到一个"视觉 token"。

例如 224×224 的图像，patch=14，则切成 `(224/14)² = 256` 个 patch，每个 patch 投影成 D 维向量，形成长度为 256 的视觉 token 序列。序列前面通常再加一个 `[CLS]` token。这些 token 经过多层 Transformer 编码后，就携带了图像的语义信息。

patch 大小是精度与成本的权衡：patch 越小，token 越多、细节越丰富，但序列变长、计算量上升。

## 模态对齐：把视觉 token 接入 LLM

视觉 token 的维度和 LLM 的词嵌入维度往往不一致，需要"对齐层"：

- **MLP Projector**：一个两层 MLP，把视觉 token 映射到文本 embedding 维度。简单高效，是最常见的拼接式做法。
- **Cross-Attention（如 Q-Former / Perceiver）**：用一组可学习的 query 通过交叉注意力从视觉特征里"抽取"固定数量的 token，能压缩视觉序列长度，并支持更灵活的交互。

对齐质量直接决定多模态效果：对齐层太弱，LLM "看不懂"视觉 token；对齐层训练不充分，会出现"看图说话但细节全错"的现象。

## 视频理解：时间维度的处理

视频比图像多一个时间轴，处理有两层关键：

1. **帧采样**：视频不能全塞进去，需按固定间隔或关键帧采样出若干帧，每帧独立过视觉编码器得到 token。采样策略决定时间分辨率——采得太稀会丢失快速动作，采得太密则序列过长。
2. **时序位置编码（3D RoPE）**：普通 RoPE 只对序列位置编码；视频需要同时编码**时间 t、高度 h、宽度 w** 三个维度。3D RoPE 把旋转位置编码扩展到三个轴，让模型感知帧的先后顺序与空间位置。

## 实操一：ViT 的 patch 切分

下面用 `einops` 演示图像到 patch token 的核心变换：

```python
import torch
from einops.layers.torch import Rearrange
import torch.nn as nn

class PatchEmbed(nn.Module):
    def __init__(self, img_size=224, patch_size=14, in_ch=3, embed_dim=768):
        super().__init__()
        n = img_size // patch_size
        self.proj = nn.Sequential(
            # 图像 -> N 个 patch，每个展平为向量
            Rearrange("b c (h p1) (w p2) -> b (h w) (c p1 p2)",
                      p1=patch_size, p2=patch_size),
            nn.Linear(in_ch * patch_size * patch_size, embed_dim),
        )
        self.num_patches = n * n

    def forward(self, img):
        # img: [B, 3, 224, 224]
        return self.proj(img)   # [B, 256, 768]

patch = PatchEmbed()
x = torch.randn(2, 3, 224, 224)
tokens = patch(x)
print(tokens.shape)   # torch.Size([2, 256, 768])
```

## 实操二：图像 → 视觉 token → 拼接文本 embedding

这是拼接式多模态的完整骨架：图像经视觉编码器和 projector 变成视觉 embedding，与文本 embedding 拼接后送入 LLM。

```python
import torch
import torch.nn as nn

class SimpleVLM(nn.Module):
    def __init__(self, visual_encoder, llm,
                 vision_dim=768, text_dim=4096):
        super().__init__()
        self.visual = visual_encoder
        # 对齐层：把视觉 token 映射到文本 embedding 维度
        self.projector = nn.Sequential(
            nn.Linear(vision_dim, text_dim),
            nn.GELU(),
            nn.Linear(text_dim, text_dim),
        )
        self.llm = llm

    def forward(self, images, input_ids, text_embeds):
        # 1) 图像编码 -> 视觉 token
        vtokens = self.visual(images)              # [B, Nv, vision_dim]
        vembeds = self.projector(vtokens)          # [B, Nv, text_dim]

        # 2) 与文本 embedding 拼接成统一序列
        #    text_embeds: [B, Nt, text_dim]
        fused = torch.cat([vembeds, text_embeds], dim=1)  # [B, Nv+Nt, text_dim]

        # 3) 送入 LLM（简化：直接把融合 embedding 交给语言模型主干）
        out = self.llm(inputs_embeds=fused)
        return out

# 要点：projector 是训练时重点，常先冻结视觉与 LLM 只训对齐层，再解冻微调
```

真实系统会插入模态专用的 `<image>` 占位符并在 prompt 中标记视觉 token 的位置，让 LLM 知道序列里哪段是图像。

## 实操三：3D RoPE 的时序位置编码

把 RoPE 扩展到时间、高、宽三个轴，是视频理解的关键实现之一：

```python
import torch

def rope_1d(x, dim, pos):
    """对最后一维做旋转。x: [B, heads, N, dim]，pos: [N] 每个 token 在该轴上的坐标。"""
    inv_freq = 1.0 / (10000 ** (torch.arange(0, dim, 2).float() / dim))
    ang = torch.einsum("n,j->nj", pos.float(), inv_freq)    # [N, dim/2]
    cos, sin = ang.cos()[None, None], ang.sin()[None, None]  # 广播到 [B, heads, N, dim/2]
    x_even, x_odd = x[..., ::2], x[..., 1::2]
    return torch.stack([x_even * cos - x_odd * sin,
                        x_even * sin + x_odd * cos], dim=-1).flatten(-2)

def rope_3d(x, pos_t, pos_h, pos_w, dim_per_axis):
    """
    x: [B, heads, N, dim]，N = T*H*W
    pos_t/pos_h/pos_w: [N]，每个 token 自己在时间/高/宽轴上的坐标
    思路：把特征维切成三段，分别用三个轴的坐标做 1D RoPE
    """
    d = dim_per_axis
    xt, xh, xw = x[..., :d], x[..., d:2*d], x[..., 2*d:]
    return torch.cat([rope_1d(xt, d, pos_t),
                      rope_1d(xh, d, pos_h),
                      rope_1d(xw, d, pos_w)], dim=-1)

# 构造坐标：N 个 token，每个 token 有自己的 (t, h, w)
T, H, W = 8, 16, 16
idx = torch.arange(T * H * W)
pos_t = idx // (H * W)
pos_h = (idx % (H * W)) // W
pos_w = idx % W
```

这里的关键是三个参数必须是**逐 token 的坐标张量**（长度为 N），而不是三个整数。传整数的话，`arange` 出来的位置和序列维对不上，广播直接报错；就算凑巧对上，也是在给所有 token 套同一套位置编码，等于没做三维扩展。

工程上更常见的是预计算三个轴的 cos/sin 表（每个轴一张 `[max_len, dim/2]`），前向时按 `(t,h,w)` 索引取用，避免逐 token 重复算 `cos/sin`。

## 多模态训练的难点

**模态冲突**：文本与图像的梯度方向和尺度不同，联合训练时容易顾此失彼，出现一种模态"压制"另一种。常见对策是分阶段训练——先冻结 LLM 只训对齐层，再逐步解冻；并为不同模态设置不同的学习率。

**数据配比**：文本、图像、视频数据的比例直接决定模型"偏科"。图像数据过多可能损伤纯文本能力，文本过多则视觉理解不足。配比是需反复实验的核心超参。

**视觉 token 序列长度**：高分辨率图像和长视频会生成极长的视觉 token 序列，显著抬升注意力成本。压缩（如 Q-Former 或 token 合并）是必然选择。

## 工程实践：用 API 还是自研

- **优先用现成 VLM API**：绝大多数应用（看图问答、文档理解、内容审核）不需要自研，直接调用成熟多模态 API 即可，开发快、效果稳。
- **考虑微调（Fine-tuning）的场景**：有领域专属图像（如医疗影像、工业质检）、数据隐私要求高、或需要定制输出格式时，可在开源 VLM 上做对齐层或 LoRA 微调——成本远低于从头预训练。
- **从头预训练原生多模态**：只有极少数具备海量数据与算力的团队才值得投入；对绝大多数团队，这是不经济的选择。

## 小结

原生多模态的工程链条可以概括为：**ViT 把图像切成 patch token，对齐层（MLP projector 或 cross-attention）把视觉 token 映射进文本 embedding 空间，视频再叠加帧采样与 3D RoPE 处理时间维度，最终多种模态在同一 token 空间里被统一建模**。拼接式方案成本低、落地快，原生方案能力上限高但代价昂贵。理解这条链路，你才能在"调 API、微调开源模型、还是从头自研"之间做出清醒的工程决策，而不是被"原生"二字牵着走。
