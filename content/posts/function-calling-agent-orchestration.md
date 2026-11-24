---
title: "拆解 Function Calling：从工具 schema 到多步 Agent 编排"
slug: "function-calling-agent-orchestration"
date: 2026-05-15
draft: false
description: "从一个工具定义到多步 Agent 循环，完整拆解 Function Calling 的链路、状态机设计与工程落地要点。"
tags: ["Agent", "Function Calling", "工具调用", "LLM"]
categories: ["后端与架构"]
aliases:
  - "/posts/拆解-function-calling从工具-schema-到多步-agent-编排/"
---

## 热点引入：从"能聊天"到"能干活"

2026 年 5 月，阿里发布 Qwen3.7 Max 等 agent 旗舰模型，宣传重心明显从"对话更自然"转向了"能自主完成多步任务"。整个行业的竞争焦点也在迁移：模型会不会聊天已经不是差异化卖点，**能不能可靠地调用工具、编排流程、把活干完**才是。

对后端工程师来说，这意味着一个新的基础设施层出现了——Agent 编排。它看起来像普通的函数调用，但涉及 schema 定义、结果回填、错误恢复、循环终止等一系列工程问题。很多团队把 demo 跑通后直接上生产，结果被"工具调用死循环""参数幻觉""结果没回填导致模型重复调用"这类问题折腾得够呛。本文就把这条链路完整拆一遍。

## 技术原理：一次工具调用的完整链路

Function Calling 本质上是让模型输出一段**结构化意图**，而不是自然语言。完整链路有五个环节：

1. **定义工具 schema**：用 JSON Schema 描述工具的名字、参数、类型与约束，随请求一起发给模型。
2. **模型返回 tool_call**：模型判断需要调用工具时，不直接回答，而是返回 `{name, arguments}` 结构。
3. **执行工具**：应用在服务端真正执行这个函数，拿到结果。注意模型本身不会执行任何代码。
4. **结果回填**：把执行结果作为 `role: "tool"` 的消息追加到对话历史。
5. **继续推理**：再次调用模型，让它基于工具结果继续推理或给出最终答案。

一个容易踩的认知坑是：**模型返回的 arguments 是它"猜"出来的 JSON**，不是类型安全的运行时对象。参数缺失、类型错误、枚举值越界都很常见，所以执行前必须做校验。

## 实操：规范的工具定义与 Agent Loop

### OpenAI 兼容格式的工具 schema

```python
import json

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查询指定城市在给定日期的天气情况。用于回答天气、气温、降水相关问题。",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，使用中文全称，如'杭州市'。"
                    },
                    "date": {
                        "type": "string",
                        "description": "日期，格式 YYYY-MM-DD。不填表示今天。"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位，默认 celsius。"
                    }
                },
                "required": ["city"],
                "additionalProperties": False,
            },
        },
    }
]
```

`description` 不是给人看的注释，而是**模型选工具、填参数的依据**。写得含糊，模型就会乱填。`additionalProperties: False` 能减少模型凭空捏造字段的概率。

### 并行调用与串行依赖

模型可能一次返回多个 `tool_call`。判断标准是**彼此是否有数据依赖**：查询北京天气和查询上海天气互不依赖，可以并行执行以降低延迟；而"先查订单号 → 再用订单号查物流"是串行依赖，第二个调用的参数来自第一个的结果，必须顺序执行。工程上通常把一次返回的无依赖调用放进线程池并发执行，再统一回填。

### Agent Loop：带保护的状态机

多步编排的核心是一个循环：调用模型 → 若有 tool_call 则执行并回填 → 直到模型给出最终答案。必须加最大步数与超时保护，否则容易陷入"调用—失败—再调用"的死循环。

```python
import json
from openai import OpenAI

client = OpenAI()

TOOL_IMPL = {
    "get_weather": lambda city, date=None, unit="celsius": {
        "city": city, "date": date or "today",
        "temp": 26, "unit": unit, "condition": "晴"
    },
}

def run_agent(user_query, max_steps=8):
    messages = [
        {"role": "system", "content": "你是助手，可调用工具。始终基于工具结果回答。"},
        {"role": "user", "content": user_query},
    ]
    for step in range(max_steps):
        resp = client.chat.completions.create(
            model="qwen3.7-max", messages=messages,
            tools=TOOLS, tool_choice="auto",
        )
        msg = resp.choices[0].message
        messages.append(msg)

        if not msg.tool_calls:
            return msg.content              # 无工具调用，给出最终答案

        # 执行本轮所有工具调用（无依赖可并行，这里简化为串行）
        for call in msg.tool_calls:
            name = call.function.name
            args = json.loads(call.function.arguments)
            try:
                result = TOOL_IMPL[name](**args)
                content = json.dumps(result, ensure_ascii=False)
            except Exception as e:
                content = json.dumps({"error": str(e)}, ensure_ascii=False)
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": content,
            })
    raise RuntimeError("超过最大步数，疑似循环调用")
```

注意每个工具消息都带 `tool_call_id`，把结果和具体的调用一一对应——并行调用时这一点至关重要，否则模型分不清哪个结果对应哪个请求。

## 工程实践建议与踩坑

- **给循环设三重保护**：最大步数、总超时、单工具超时。缺一都可能让一次请求永久卡住。
- **参数校验放在执行层**：模型输出的 arguments 过不了 Pydantic 校验时，把明确的错误信息回填给模型，让它自我纠正，往往比直接抛异常更有效。
- **重试要区分错误类型**：参数错误重试无意义（模型可能重复犯错），应回填错误让模型修正；网络超时、限流这类瞬时错误才适合指数退避重试。
- **工具描述写清楚三件事**：这个工具**做什么**、**什么时候该用/不该用**、**每个参数的格式与语义**。尤其是边界条件（例如"用户没指定日期时传今天"）要写明。
- **控制工具数量**：一次性塞给模型几十个工具会显著降低选择准确率，可以按场景动态注入相关工具。
- **幂等性**：工具可能被重复调用，涉及写操作的工具要靠幂等键避免重复扣款、重复下单。

## 小结

Function Calling 的工程难点不在"让模型调一次工具"，而在**把它当成一个不可靠但可纠错的协作者来设计**。清晰的 schema 是输入质量的基础，严格的参数校验是安全底线，带保护的循环状态机是稳定运行的保障，而结构化的错误回填则让模型具备自我修复能力。把这几层做好，"能干活"的 Agent 才从 demo 走向生产。
