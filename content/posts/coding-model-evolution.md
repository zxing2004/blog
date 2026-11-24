---
title: "编码模型的下一站：从补全到自主 Agent 的架构演进"
date: 2026-11-17
draft: false
description: "编码模型正从单行补全走向自主 Agent。本文梳理补全、对话编辑、Agent 三阶段架构，讲透 FIM、上下文构建与 IDE 集成，并给出 tree-sitter 实操。"
tags: ["编码模型", "Copilot", "Agent", "IDE"]
categories: ["工具与效率"]
---

## 热点引入：编码模型的重心迁移

2026 年下半年，微软把自研编码模型接入了 GitHub Copilot，几家头部厂商的新一代编码模型也几乎同期亮相。具体型号与版本号几个月就翻一茬，本文不逐个点名（以各厂商官方发布为准），因为它们的具体能力差异并不是重点。把这几条消息放在一起看，一条主线格外清晰：编码模型的竞争重心，正从「补全得准不准」转向「能不能自己把一个任务做完」。

回溯这几年，编码助手的交互形态经历了明显跃迁。最早的 Copilot 是「你写半行，它补半行」；后来有了聊天窗口，可以描述需求让它改一段代码；再到如今，你丢给它一个 issue，它自己读仓库、跨文件修改、跑测试、提交 PR。能力叙事在升级，但支撑它们的工程架构完全不同。

理解这场演进，对工程师很有价值：它决定了我们该在什么场景用什么形态，也决定了构建一个编码工具时，底座要怎么设计。本文按三个阶段拆解——补全、对话式编辑、自主 Agent，并重点讲透 FIM 训练格式、Agent 的上下文构建与 IDE 集成这三类硬骨头。

## 技术原理：三个阶段的架构差异

**第一阶段：单行/块补全与 FIM。** 朴素的补全只看光标上文（prefix），这有个致命缺陷：它不知道光标后面写了什么，容易补出与后文重复或冲突的代码。**FIM（Fill-in-the-Middle，中间填充）** 解决了这个问题——模型同时看 prefix 和 suffix，学会在中间「填空」。

FIM 的训练格式关键是把文档改造成特殊序列。常见做法（以字符级 FIM 为例）：

```
<fim_prefix> + prefix + <fim_suffix> + suffix + <fim_middle> + middle
```

模型在 `<fim_middle>` 之后预测 `middle`（即光标处应填的内容）。推理时把真实 prefix/suffix 套进模板，让模型生成中间部分。构造 prompt 时要注意：prefix 通常取光标前若干行，suffix 取光标后若干行，并用特殊 token 显式分隔；还要处理 token 边界对齐，避免把 suffix 的第一个 token 切碎。

```python
FIM_PREFIX = "<fim_prefix>"
FIM_SUFFIX = "<fim_suffix>"
FIM_MIDDLE = "<fim_middle>"

def build_fim_prompt(text: str, cursor: int, max_prefix: int = 2048,
                     max_suffix: int = 1024) -> str:
    """text 为当前文件全文，cursor 为光标字符偏移。"""
    prefix = text[max(0, cursor - max_prefix):cursor]
    suffix = text[cursor:cursor + max_suffix]
    # prefix/suffix 放入各自的特殊段，最后以 <fim_middle> 收尾触发生成
    return f"{FIM_PREFIX}{prefix}{FIM_SUFFIX}{suffix}{FIM_MIDDLE}"

src = "def add(a, b):\n    # 在这里实现\n    \nprint(add(1, 2))"
prompt = build_fim_prompt(src, src.index("    \n") + 4)
# 模型接着 prompt 生成 "return a + b" 即可完成填空
```

**第二阶段：对话式编辑。** 交互从「填空」变成「描述意图 → 改代码」。工程上有两种落地形态：一是让模型输出**统一 diff / patch**，由编辑器应用；二是生成**内联 diff**，在代码旁直接显示增删行让用户逐块接受。这里的关键问题是编辑冲突：用户可能在模型生成期间又改了文件，此时 patch 的上下文行号已失效。解决办法是基于内容（而非行号）匹配、生成带足够上下文的 patch，并在应用前校验文件哈希。

**第三阶段：自主 Agent。** 模型不再只改一个文件，而是理解整个仓库、调用工具、执行多步长任务。核心变化是**上下文构建**：窗口装不下整个仓库，必须检索出相关片段。典型管线是——先建仓库索引（符号表、向量索引），再根据任务检索相关文件，用 **AST 分块**把大文件切成语义完整的片段（函数、类），最后按 token 预算拼装上下文。工具调用（读文件、搜索、跑测试、执行命令）则让 Agent 能真正闭环验证。

## 实操：用 tree-sitter 做符号提取与 AST 分块

为什么用 AST 分块而不是按行数硬切？因为行切块会把一个函数劈成两半，破坏语义，模型看不懂。tree-sitter 能解析出语法树，按函数/类边界切分，保留完整结构。

```python
from tree_sitter import Language, Parser
import tree_sitter_python as tspy

PY_LANGUAGE = Language(tspy.language())
parser = Parser(PY_LANGUAGE)

def extract_symbols(source: bytes) -> list[dict]:
    """提取 Python 源码中的顶层函数与类及其字节范围。"""
    tree = parser.parse(source)
    root = tree.root_node
    symbols = []
    for node in root.named_children:
        if node.type in ("function_definition", "class_definition"):
            name_node = node.child_by_field_name("name")
            symbols.append({
                "type": node.type,
                "name": name_node.text.decode() if name_node else "<anon>",
                "start": node.start_byte,
                "end": node.end_byte,
                "code": source[node.start_byte:node.end_byte].decode(),
            })
    return symbols

source = open("service.py", "rb").read()
for sym in extract_symbols(source):
    print(f"{sym['type']:20s} {sym['name']:30s} {sym['end']-sym['start']} bytes")
```

有了符号粒度，就能做「相关文件检索 + 符号级上下文裁剪」：先定位任务涉及的文件，再在文件内挑选符号相关的块，精确控制 token 预算。这比整文件塞进窗口高效得多。

IDE 集成是另一处工程重点。**LSP（Language Server Protocol）** 提供符号跳转、引用查找、诊断等能力，Agent 可借此构建更准的上下文；编辑应用则要处理并发——用户光标处正在输入时，不要直接覆盖，而应采用增量应用 + 冲突检测。

```python
def can_apply_patch(file_text: str, expected_hash: str) -> bool:
    """应用 patch 前校验文件内容未变，否则拒绝，避免覆盖用户改动。"""
    import hashlib
    return hashlib.sha256(file_text.encode()).hexdigest() == expected_hash

if can_apply_patch(current_text, patch.base_hash):
    apply_patch(current_text, patch)
else:
    # 回退：让用户确认，或基于新内容重新生成 patch
    raise ConflictError("文件已被修改，需要重新生成补丁")
```

## 工程实践建议与趋势

**何时用补全、何时用 Agent，要有明确决策。** 高频、局部、意图清晰的改动（补一行、写个函数体）用补全/FIM，延迟低、打断少；跨文件、需要理解业务上下文、涉及测试验证的任务才交给 Agent，因为它慢且可能出错。中间地带用对话式编辑。一个实用原则是「能补不全用 Agent」，把重武器留到真正需要的地方。

**必须设计 fallback 策略。** Agent 任务失败或超时时，要能优雅降级：保留已产生的 diff 供人工接手、回滚到干净工作区、给出失败原因。永远不要在后台悄悄改一堆文件却让用户不知情。此外，所有自动修改都应纳入版本控制，便于 diff 审查和一键撤销。

**可观测性不能缺。** 记录 Agent 每一步的工具调用、上下文快照和耗时，才能分析「它为什么改错」。此前讨论过的循环检测在这里同样适用——Agent 反复读写同一文件却无进展时，应及时熔断。

## 小结

编码模型的演进，本质是交互粒度与上下文规模的双重升级：从只看 prefix 的补全，到 prefix/suffix 双向的 FIM，再到带 diff 应用的对话编辑，最终走向具备仓库理解与工具调用能力的自主 Agent。支撑这场跃迁的，是 FIM 训练格式、AST 级上下文构建、LSP 与冲突处理这些扎实工程。工具越来越强，但选择权仍在我们手里——按任务复杂度匹配合适的形态，配上 fallback 与可观测性，才能真正把编码模型变成可靠的生产力，而不是制造混乱的「自动化黑洞」。
