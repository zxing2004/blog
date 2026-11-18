---
title: "百万上下文时代：RAG 还是长上下文？一份工程决策指南"
date: 2026-05-08
draft: false
description: "当百万 token 上下文成为标配，RAG 是否还有必要？本文从原理、成本、准确性到可追溯性给出一套可落地的工程决策框架。"
tags: ["RAG", "长上下文", "LLM", "架构"]
categories: ["后端与架构"]
---

## 热点引入：百万上下文成为标配之后

2026 年 5 月的 Google I/O 上，Gemini 3.5 Flash 把上下文窗口推到了 100 万 token，"百万上下文成为标配"这句话从营销口号变成了摆在架构师面前的一道真实选择题。几乎同一时间，主流闭源与开源模型纷纷把上下文长度往上堆，似乎"把整个知识库塞进提示词"已经指日可待。

于是团队里开始出现一种声音：既然模型能一次读下几百页文档，是不是可以砍掉那套维护了半年的 RAG 管线？检索、切块、向量库、重排序——这些"中间层"是不是历史包袱了？

这个判断不能说全错，但过于乐观。上下文窗口变大解决的是"能不能装下"，却没有解决"装下之后模型能不能用得好、用得起、用得可信"。本文就从这三个维度拆开讲，并给出一份可以拿去开会的决策表格。

## 技术原理：长上下文的两个"隐性税"

### 上下文腐烂（Context Rot）

上下文长度增加并不等于有效信息量线性增加。研究者观察到一种被称为**上下文腐烂**的现象：随着无关内容增多，模型对关键信息的注意力被稀释，回答质量反而下降。这有点像一个人在堆满杂物的房间里找钥匙——房间越大，翻找成本越高。长上下文需要处理海量的 token 间关系，注意力被平摊到大量噪声上，核心指令的权重被拉低。

### 迷失在中间（Lost in the Middle）

另一个被反复验证的现象是**位置偏差**：模型对位于上下文开头和结尾的信息 recall 较高，而对"夹在中间"的段落明显更容易忽略。当我们把几十篇文档拼成一条超长提示词时，真正命中的那一段很可能恰好落在中间区域，于是被静默地忽略了。这不是 prompt 写法能完全绕开的，而是当前 Transformer 注意力分布的结构性特征。

### 成本结构：注意力 O(n²)，KV Cache O(n)

长上下文的代价是实打实的。自注意力的计算复杂度随序列长度呈平方增长，这意味着序列长度翻倍，注意力计算量约变为四倍。而推理阶段缓存的 KV Cache 显存占用随序列长度线性增长：每个 token 的每一层都要保存 Key 和 Value 两个向量，显存大致正比于 `层数 × 序列长度 × 隐藏维度 × 精度字节数 × 2`。100 万 token 的 KV Cache 本身就可能吃掉数十 GB 显存，这也是为什么长上下文调用往往"首 token 延迟"很高、单价更贵。

下面这段代码估算 KV Cache 显存，方便你直观感受量级：

```python
def kv_cache_gb(seq_len, layers=32, hidden=4096, bytes_per_elem=2, batch=1):
    """估算 KV Cache 显存（GB）。fp16 用 2 字节，int8 用 1 字节。"""
    # 每层每 token 需保存 K 和 V 两个 hidden 向量
    per_token = layers * hidden * 2 * bytes_per_elem
    total_bytes = batch * seq_len * per_token
    return total_bytes / (1024 ** 3)

for n in (8_000, 32_000, 128_000, 1_000_000):
    print(f"{n:>9,} tokens -> {kv_cache_gb(n):6.2f} GB")
```

可以看到，从 8K 到 1M，显存占用是百倍级增长。这就是"能装下"和"装得起"之间的鸿沟。

## RAG 与长上下文：逐项对比

| 维度 | RAG | 长上下文 |
| --- | --- | --- |
| 单次调用成本 | 低（只送入召回片段） | 高（全文入模，KV Cache 贵） |
| 首 token 延迟 | 低 | 高（预填充海量 token） |
| 准确性 | 依赖召回质量，命中片段小、干扰少 | 受上下文腐烂与迷失在中间影响 |
| 可追溯性 | 强，天然带来源文档与片段 | 弱，难以定位依据来自哪一段 |
| 文档更新 | 增量重建索引即可 | 每次都要重新组装提示词 |
| 适用规模 | 大规模、高频更新知识库 | 中小规模、一次性精读 |

结论很清晰：**长上下文擅长"精读少量文档"，RAG 擅长"从上万篇里找到相关几篇"**。二者不是替代关系，而是可以组合。

## 实操：语义分块与混合检索

### 基于 token 的语义分块

固定字符切分会切断句子和段落，语义分块则按标题、段落边界切分，并用 token 计数而非字符数来控制块大小（因为模型的窗口是按 token 算的）。

```python
from tiktoken import encoding_for_model

enc = encoding_for_model("gpt-4o")

def token_len(text: str) -> int:
    return len(enc.encode(text))

def semantic_chunk(text: str, max_tokens: int = 512, overlap: int = 64):
    """按段落边界切分，尽量不切断语义单元。"""
    paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]
    chunks, buf, buf_len = [], [], 0
    for p in paragraphs:
        plen = token_len(p)
        if buf_len + plen > max_tokens and buf:
            chunks.append("\n\n".join(buf))
            # 保留尾部一段作为重叠，缓解边界信息丢失
            tail, tail_len = [], 0
            for sent in reversed(buf):
                sl = token_len(sent)
                if tail_len + sl > overlap:
                    break
                tail.insert(0, sent)
                tail_len += sl
            buf, buf_len = tail, tail_len
        buf.append(p)
        buf_len += plen
    if buf:
        chunks.append("\n\n".join(buf))
    return chunks
```

关键点是用 `token_len` 而不是 `len(text)` 来控制块大小，否则中文或代码场景下字符数与 token 数可能相差一倍以上，导致块实际超长。

### BM25 + 向量的混合检索（骨架）

混合检索把关键词精确匹配（BM25）和语义相似度（向量）的结果融合，兼顾专有名词与语义泛化：

```python
def hybrid_search(query, bm25_index, vector_index, top_k=10, alpha=0.5):
    # 两路各自召回
    bm25_hits = bm25_index.search(query, top_k * 2)   # [(doc_id, score), ...]
    q_vec = embed(query)
    vec_hits = vector_index.search(q_vec, top_k * 2)

    # 归一化到 [0,1] 后加权融合（RRF 也是常见选择）
    def normalize(hits):
        scores = [s for _, s in hits]
        lo, hi = min(scores), max(scores)
        return {d: (s - lo) / (hi - lo + 1e-9) for d, s in hits}

    nb, nv = normalize(bm25_hits), normalize(vec_hits)
    fused = {}
    ids = set(nb) | set(nv)
    for d in ids:
        fused[d] = alpha * nb.get(d, 0) + (1 - alpha) * nv.get(d, 0)
    return sorted(fused.items(), key=lambda x: -x[1])[:top_k]
```

## 工程实践建议与踩坑

- **不要全盘砍掉 RAG**。更稳妥的混合架构是"RAG 粗筛 + 长上下文精读"：先用检索从上万篇里召回几十个候选，再把这几十个片段一次性交给长上下文模型做交叉验证与精读。这样既利用了长上下文的全局理解能力，又避开了全文入模的成本。
- **注意位置摆放**。把最关键的材料放在提示词首尾，缓解迷失在中间；重要指令尽量前置。
- **重叠块要去重**。语义分块的重叠会让同一内容出现多次，召回后需要按文档 ID + 段落做去重，否则会挤占上下文。
- **成本要观测**。上线后记录每次调用的输入 token 数，长上下文调用的成本可能是普通调用的几十倍，不做预算很容易失控。
- **可追溯性是刚需场景的硬指标**。法律、医疗、金融问答必须能给出引用来源，这类场景 RAG 几乎不可替代。

## 小结

百万上下文不是 RAG 的丧钟，而是一件新工具。判断标准可以归结为一句话：**文档规模小、一次性精读、不要求引用溯源时，长上下文更省事；规模大、更新频繁、需要溯源和成本可控时，RAG 仍是主力**。成熟的工程方案往往是两者混合——让长上下文做粗筛与精读，让 RAG 负责召回与溯源。理解各自的成本曲线和能力边界，比追逐窗口数字更重要。
