---
title: "模型路由实战：按任务分层调度 Sol/Terra/Luna 类模型"
date: 2026-07-16
draft: false
description: "当同一平台提供旗舰、均衡、廉价多档模型，如何把每个请求送到性价比最高的那一档？本文讲清规则、分类器与级联三类路由策略，并给出可运行的路由器与成本统计实现。"
tags: ["模型路由", "成本优化", "LLM", "调度"]
categories: ["工具与效率"]
---

## 热点引入：GPT-5.6 家族发布，"选对模型"成了一门学问

2026 年 7 月，OpenAI 发布 GPT-5.6 家族，首次以"模型家族"形态亮相：Sol 定位旗舰、追求最强能力；Terra 走均衡路线；Luna 则主打快速与廉价。三档模型能力定性地依次递减，价格却相差数倍。

这一发布把一个问题推到所有工程团队面前：**你的应用真的需要每个请求都调用最贵的那档吗？** 真实业务的请求难度分布极不均匀——大量是简单分类、格式转换、摘要；少数是复杂推理、长代码生成。如果一律用旗舰模型，等于用 F1 赛车送外卖。模型路由（Model Routing）就是解决这件事：在请求进入时判断其难度，把它分派到恰到好处的模型档位。

路由不是简单的省钱技巧，它是一门需要在成本、延迟、质量三角中做动态权衡的工程。本文给出一套从策略到代码再到度量的完整落地路径。

## 技术原理：三类路由策略

在动手前，先理解三种主流策略及其适用面。

**规则路由**：用手工规则判断，例如按输入字符数、任务类型标签、是否包含代码等特征分档。它零成本、可解释、易调试，缺点是规则粗糙，边界情况处理生硬。适合作为基线或兜底层。

**分类器路由**：训练或调用一个小模型（甚至一个 embedding + 线性层）预测请求的难度或"该用哪一档"。它能捕捉语义特征，泛化能力远强于硬编码规则。代价是要额外维护一个分类器，且自身也有推理开销与误判风险。

**级联路由（Cascade）**：先用便宜模型尝试，根据其输出的置信度决定是否"升级"到更强的模型。它本质是一种串行赌注——多数请求在第一级就被廉价满足，难题才付出升级成本。关键在置信度校准：便宜模型"自信地答错"是级联最大的敌人。

三者并不互斥。生产系统常见组合是：**规则兜底 + 分类器主路由 + 级联处理边界案例**。

## 实操：一个带置信度与 fallback 的路由器

下面实现一个路由器：先用规则做快速判定，不确定的请求交给分类器打分，低于置信度阈值则升级到强模型兜底。价格常量用示意值表示，替换为你的实际账单即可。

```python
from dataclasses import dataclass

# 示意单价（每百万 token），请替换为实际账单
PRICE = {"sol": 60.0, "terra": 12.0, "luna": 1.5}

@dataclass
class RouteDecision:
    model: str
    reason: str
    confidence: float

class SimpleDifficultyClassifier:
    """示意：用长度与关键词启发式模拟一个难度打分器。
    生产中应替换为训练好的分类模型或 embedding + 线性层。"""
    def predict(self, text: str) -> tuple[str, float]:
        hard_kw = ("证明", "重构", "架构", "推导", "debug")
        easy_kw = ("翻译", "分类", "摘要", "格式化")
        if any(k in text for k in easy_kw) and len(text) < 500:
            return "luna", 0.90
        if any(k in text for k in hard_kw) or len(text) > 4000:
            return "sol", 0.80
        return "terra", 0.55          # 不确定，置信度低

class ModelRouter:
    def __init__(self, classifier, conf_threshold=0.75):
        self.classifier = classifier
        self.conf_threshold = conf_threshold

    def route(self, text: str) -> RouteDecision:
        # 规则层：超长输入直接走旗舰，避免分类器误判
        if len(text) > 8000:
            return RouteDecision("sol", "rule:超长输入", 1.0)

        target, conf = self.classifier.predict(text)
        # 置信度不足则升级到更强一档兜底
        if conf < self.conf_threshold:
            target = self._escalate(target)
            return RouteDecision(target, f"fallback:低置信度{conf:.2f}", conf)
        return RouteDecision(target, "classifier", conf)

    @staticmethod
    def _escalate(model: str) -> str:
        return {"luna": "terra", "terra": "sol", "sol": "sol"}[model]

router = ModelRouter(SimpleDifficultyClassifier())
print(router.route("把这段话翻译成英文"))       # -> luna
print(router.route("帮我重构这个分布式模块"))   # -> sol
```

`confidence` 是这里的核心：它让路由器能识别"我不确定"，从而把难题交给强模型，而不是赌一把。没有置信度的路由是脆弱的。

## A/B 实验与成本统计

上线路由前必须用 A/B 验证。下面给出分桶与成本统计的最小实现，把"省了多少钱、质量掉没掉"变成可度量的数字。

```python
import hashlib

PRICE = {"sol": 60.0, "terra": 12.0, "luna": 1.5}  # 每百万 token 示意单价

class ABTracker:
    def __init__(self, salt="exp-2026-07"):
        self.salt = salt
        self.records = []

    def bucket(self, user_id: str) -> str:
        """按用户稳定分桶，保证同一用户始终在同一组"""
        h = int(hashlib.md5(f"{self.salt}:{user_id}".encode()).hexdigest(), 16)
        return "control" if h % 2 == 0 else "treatment"

    def record(self, group, model, in_tok, out_tok, quality_score):
        cost = (in_tok + out_tok) / 1e6 * PRICE[model]
        self.records.append((group, model, cost, quality_score))

    def report(self):
        for group in ("control", "treatment"):
            rows = [r for r in self.records if r[0] == group]
            total_cost = sum(r[2] for r in rows)
            avg_quality = sum(r[3] for r in rows) / len(rows)
            print(f"{group}: 成本={total_cost:.2f} 平均质量={avg_quality:.3f}")

# control 组全部走 sol；treatment 组启用路由
tracker = ABTracker()
tracker.record("control", "sol", 1000, 500, 0.93)
tracker.record("treatment", "luna", 1000, 300, 0.88)
tracker.report()
```

评判标准应当同时看两条线：**成本下降幅度**是否显著，以及**质量指标**（人工评分、任务成功率、用户满意度）是否在可接受范围内。只有成本降了、质量没实质退化，路由才算成功。

## 质量降级的风险与兜底

路由的本质是"用低档模型赌简单任务"，赌错就会退化。必须预先设计兜底：

- **级联回滚**：廉价模型输出后，若置信度低或自检失败（如格式不符合 schema、明显答非所问），立即用强模型重跑。
- **用户反馈触发**：用户点"不满意"或重试时，把该请求升级到强档，并把样本纳入分类器再训练。
- **影子流量监控**：小比例请求同时跑路由档与旗舰档，持续对比两者质量差异，作为路由健康度的金丝雀信号。
- **硬底线**：涉及合规、安全、资金的关键链路，直接禁用低档模型，不做路由。

## 工程实践建议与踩坑

- **先建基线再谈优化**。先全部走旗舰跑一段时间，拿到质量基线，才知道路由的"代价"是多少。
- **置信度要校准**。分类器的 softmax 分数往往过于自信；用温度缩放或独立校准集，让"低置信度"真正对应高风险样本。
- **缓存分类结果**。同一 prompt 模板的分类结果可缓存，避免每个请求都付一次分类开销。
- **警惕分布漂移**。业务流量随季节、活动变化，昨天"简单"的请求明天可能变难；定期用新样本评估并迭代分类器。
- **把价格当配置而非常量**。模型会降价、新档位会上线，价格表应可热更新，否则账单核算会失真。

## 小结

模型路由把"用什么模型"从一个全局配置，变成一个按请求决策的动态问题。规则路由提供可解释的骨架，分类器路由提供语义泛化，级联路由用置信度在成本与质量间做精细赌注——三者组合，配合 A/B 度量与质量兜底，才能既省下账单，又不牺牲体验。记住：路由的目标从来不是"用最便宜的模型"，而是**在可接受的质量下滑范围内，让每一分钱都花在刀刃上**。
