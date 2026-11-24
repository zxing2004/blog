---
title: "长文档处理流水线：切片、检索与摘要的工程实践"
date: 2026-06-22
draft: false
description: "从语义分块到层级摘要再到检索增强，拆解一条可落地的长文档处理流水线，并覆盖表格、代码等特殊内容。"
tags: ["长文档", "RAG", "摘要", "检索"]
categories: ["后端与架构"]
---

## 热点引入：长文档处理真正开始落地

2026 年 6 月，Claude Sonnet 5 把百万上下文下放到标准定价层，长文档处理的成本大幅下降。此前很多团队不是不想做长文档问答，而是算不过账——把几百页文档塞进模型，单次调用成本让 ROI 无从谈起。定价下探让这件事从"技术可行"走向"工程划算"。

但成本下降不等于问题消失。把整本手册丢给模型，依然会遇到注意力稀释、引用不准、响应缓慢的问题。真正决定效果的，是**文档进入模型之前的加工流水线**：怎么切、怎么摘要、怎么检索。本文就按这条链路逐段拆解。

## 技术原理：现代长文档流水线的三段式

一条稳健的长文档流水线通常包含三段：

**① 语义分块**。按标题、段落、句子等自然边界切分，而不是每 N 个字符硬切。硬切会割裂语义——把一个表格、一段论证劈成两半，检索时两边都成了残句。语义分块追求"块内语义完整、块间边界清晰"。

**② 层级摘要（map-reduce 摘要树）**。对超长文档，先对每个块分别摘要（map），再对摘要汇总二次摘要（reduce），形成"块摘要 → 章节摘要 → 全文摘要"的树状结构。查询时既可以看全貌，也可以下钻到具体块。

**③ 检索增强**。用户提问时，先用检索定位相关块，再把少量高相关块送入模型作答，避免全文入模。检索链路里**重排序（rerank）**是关键一环：向量检索召回 Top 30 后，用交叉编码器精排，挑出真正相关的 Top 5，显著提升精度。

## 实操：切分、重排与层级摘要

### 按 token 的递归字符切分器

```python
from tiktoken import encoding_for_model
enc = encoding_for_model("gpt-4o")

def split_by_tokens(text, max_tokens=512, overlap=50, separators=None):
    """按分隔符优先级递归切分：某一级切出的子块若仍超长，就降一级继续切。"""
    separators = separators or ["\n\n", "\n", "。", "；", " ", ""]
    if len(enc.encode(text)) <= max_tokens:
        return [text]

    sep, rest = separators[0], separators[1:]

    # 这一级分隔符用不上（或已到最后一级）：换下一级；没有下一级就按 token 硬切
    if sep == "" or sep not in text:
        if not rest:
            toks = enc.encode(text)
            step = max_tokens - overlap
            return [enc.decode(toks[i:i + max_tokens])
                    for i in range(0, len(toks), step)]
        return split_by_tokens(text, max_tokens, overlap, rest)

    chunks, cur = [], ""
    for part in text.split(sep):
        candidate = cur + (sep if cur else "") + part
        if len(enc.encode(candidate)) <= max_tokens:
            cur = candidate
            continue
        if cur:
            chunks.append(cur)
            # 重叠部分整段 decode 再拼，逐 token decode 会把中文和子词切碎
            tail = enc.decode(enc.encode(cur)[-overlap:])
            cur = tail + sep + part
        else:
            cur = part
        # 单个 part 本身就超长：带着下一级分隔符递归切开
        if len(enc.encode(cur)) > max_tokens and rest:
            sub = split_by_tokens(cur, max_tokens, overlap, rest)
            chunks.extend(sub[:-1])
            cur = sub[-1]
    if cur:
        chunks.append(cur)
    return chunks
```

真正的递归体现在两处：一是这一级分隔符切出来的块还超长时，带着 `rest` 再切一次；二是这一段里根本没有该分隔符时，直接降级到下一级。**只切一遍就 return 的写法不叫递归**——那会把一个超长段落原样塞进一个块，然后在送入模型时才炸出超长错误。

另一个容易忽略的细节是重叠区的处理：`enc.decode(enc.encode(cur)[-overlap:])` 是整段解码，而 `" ".join(enc.decode([t]) for t in ...)` 是逐个 token 解码再拼空格，后者会把中文和子词切得七零八落，重叠区变成乱码。

### 重排序（rerank）

```python
def retrieve_with_rerank(query, vector_index, reranker, top_k=5):
    # 粗排：向量召回更大的候选集
    candidates = vector_index.search(embed(query), top_k=30)
    texts = [c.text for c in candidates]
    # 精排：交叉编码器联合编码 query 与 doc，输出相关性分数
    scores = reranker.score(query, texts)      # 形状 [30]
    ranked = sorted(zip(candidates, scores), key=lambda x: -x[1])
    return [c for c, _ in ranked[:top_k]]
```

向量检索是"双塔"结构，查询与文档独立编码，快但粗糙；rerank 用交叉编码器把 query 和 doc 拼在一起编码，精度高但慢，所以只用在粗排后的小候选集上——这是典型的"粗排召回 + 精排排序"两段式。

### 层级摘要 map-reduce

```python
def hierarchical_summarize(chunks, summarize_fn, group_size=8):
    """先逐块摘要，再分组归并，形成两层摘要树。"""
    # map：块级摘要
    leaf = [summarize_fn(c) for c in chunks]
    # reduce：分组二次摘要
    level2 = []
    for i in range(0, len(leaf), group_size):
        group = leaf[i:i + group_size]
        level2.append(summarize_fn("\n".join(group)))
    # 继续归并直到收缩为一篇（可递归，这里示意一层）
    root = summarize_fn("\n".join(level2))
    return {"leaf": leaf, "level2": level2, "root": root}
```

`summarize_fn` 是一次 LLM 调用。摘要树的价值在于：全文摘要回答宏观问题，块摘要支持精确下钻，两者结合兼顾广度与精度。

## 特殊内容：表格与代码块

长文档里最难处理的是**表格和代码**，它们一旦被切开就彻底失去意义。工程上有三条经验：

- **表格整块保留**：识别到 Markdown/HTML 表格后作为一个不可分割的块，超长时转述为键值对列表，而不是按行切开。
- **代码块绑定上下文**：一个代码块要和它前面的说明文字、后面的解释放在一起，否则检索到代码却不知道它解决什么问题。
- **结构化元数据**：给每个块打上 `doc_id`、`section_title`、`page` 等元数据，检索时用于过滤和溯源。

## 流水线架构与工程建议

一条完整的技术文档问答流水线可以这样组织：

```
文档入库：
  原始文档 → 解析(保留表格/代码) → 语义分块(按 token)
         → 打元数据 → 生成层级摘要 → 向量化入索引

在线问答：
  用户提问 → 查询改写 → 向量粗召回(Top30)
         → rerank 精排(Top5) → 组装提示词(块+摘要)
         → LLM 生成(带引用) → 返回答案+来源
```

工程建议：

- **分块大小要实验**。太小丢失上下文，太大引入噪声，512~1024 token 常作为起点，再按业务调。
- **摘要要定期更新**。文档变更后，受影响的块摘要与上层摘要需要级联重算，否则会出现"摘要与新内容对不上"。
- **给答案挂来源**。每个块带 `doc_id` 与页码，回答时附上引用，用户可核验，也便于排查检索错误。
- **rerank 是性价比最高的精度提升点**。在召回质量尚可但精度不够时，优先加 rerank，而不是盲目扩大上下文。
- **查询改写与多路召回**。用户提问往往口语化、信息不全，直接拿原话检索容易漏召回。可以先让模型把问题改写成多个更规范的检索式，再分别走关键词与向量两路召回后合并去重。对于含缩写、产品代号或专有名词的技术问题，这一招能明显提升命中率。
- **分离"事实检索"与"生成"两步**。先把检索到的原文块原样返回给用户核验，再让模型基于这些块作答，不要在一开始就让模型自由发挥。这样即使生成出现偏差，也能快速定位是召回错了还是表达错了，排障成本大幅降低。

## 小结

长文档处理的成本下降，把瓶颈从"能不能做"转移到了"做得好不好"。一条可靠的流水线 = 语义分块保完整 + 层级摘要兼顾宏观与细节 + 检索与 rerank 保证精度。三者各司其职，再辅以表格代码的特殊处理和可溯源的元数据，才能把"百万上下文"真正变成可用、可信、可维护的生产系统，而不是一次昂贵的暴力粘贴。
