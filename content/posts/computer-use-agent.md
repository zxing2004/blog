---
title: "Computer Use Agent 的工程难点：屏幕理解与动作空间"
date: 2026-10-16
draft: false
description: "GUI Agent 从演示走向生产，难在屏幕理解、动作空间设计与容错可观测性。本文拆解 CUA 闭环，并用 Playwright 给出可运行的最小实现与循环检测方案。"
tags: ["Computer Use", "GUI Agent", "Agent", "自动化"]
categories: ["工具与效率"]
---

## 热点引入：从「看懂屏幕」到「动手操作」

2026 年 9 月，上海创智学院开源了 Nex-N2.5 系列模型，主打「所见即行动」的电脑操控能力（Computer Use Agent，CUA），一度登上 OpenRouter 热门榜首。11 月，Nex-N2.6 进一步降低了长任务中的复读率——也就是 Agent 反复执行同一无效动作、卡在原地的现象。短短两个月，GUI Agent 从论文里的炫技演示，变成了开发者手里真想接进工作流的工具。

为什么现在值得认真讨论它的工程问题？因为「能点对一次」和「能稳定干完一百步」是两回事。前者是模型能力问题，后者是系统设计问题。屏幕分辨率会变、界面会改版、弹窗会突然出现、网络会抖动——真实桌面环境的不确定性，远比聊天框里的一问一答复杂。

本文不谈模型有多强，而是拆解一个 CUA 系统必须解决的三个工程难点：屏幕理解、动作空间设计、容错与可观测性，并给出可直接改造落地的最小实现。

## 技术原理：CUA 的闭环与三道难关

所有 Computer Use Agent 都跑在同一个闭环上：**截图 → 屏幕理解 → 规划 → 执行动作 → 观察新状态 → 继续**。每一步的观察都来自屏幕，每一步的动作都落在屏幕。这个「看—想—动」循环看似简单，工程上却处处是坑。

**第一关是屏幕理解。** 模型需要把一张像素图翻译成「这里有个按钮、那里是个输入框」。纯视觉方案依赖视觉 grounding：模型输出目标元素的坐标。但坐标会受分辨率、缩放、DPI 影响，一个在 1920×1080 上正确的像素点，换到 2560×1440 就偏了。更稳妥的做法是利用可访问性树（a11y tree）：浏览器和操作系统会暴露界面的语义结构——角色、名称、层级、区域。a11y tree 天然给出语义和边界框，不依赖模型「看图猜位置」。理想策略是二者融合：先用 a11y tree 锁定候选元素，再用视觉做校验和回退。

**第二关是动作空间设计。** 动作不是越多越好，关键是坐标系与粒度。最常用的是 `click / type / scroll / key` 四件套：点击需要 (x, y) 坐标，输入需要文本与焦点位置，滚动需要方向和幅度，按键需要组合键描述。坐标系必须统一——建议对外暴露 0–1000 的归一化坐标，由适配层按当前视口换算成真实像素，这样同一份模型输出能跨分辨率复用。粒度上，应尽量使用语义化动作（如「点击名为『提交』的按钮」）而非裸坐标，把像素细节留在最底层。

**第三关是容错与可观测性。** GUI 环境充满意外：动作没生效、页面没加载、误触弹窗。系统必须能检测失败、识别循环，并保留完整轨迹供回放。没有轨迹回放，线上出问题时你根本无从复现——因为用户的屏幕状态你无法重来。

## 实操：用 Playwright 实现最小 GUI Agent 循环

下面给出一个可运行的最小骨架。核心思路：截图后调用视觉模型返回目标元素的归一化坐标，再换算成真实像素执行点击。模型调用部分用伪代码接口表示，便于替换成任意多模态模型。

```python
import base64
from playwright.sync_api import sync_playwright

# 对外统一用 0~1000 的归一化坐标系，屏蔽分辨率差异
def denormalize(nx: int, ny: int, width: int, height: int) -> tuple[int, int]:
    return int(nx / 1000 * width), int(ny / 1000 * height)

def screenshot_b64(page) -> str:
    return base64.b64encode(page.screenshot()).decode("utf-8")

def locate_target(image_b64: str, instruction: str) -> tuple[int, int]:
    """调用视觉模型，返回目标元素的归一化坐标 (nx, ny)。
    实际实现中把 image_b64 与 instruction 发给多模态模型，
    要求其只输出 JSON: {"nx": int, "ny": int}，再做解析校验。
    """
    raise NotImplementedError("接入你的多模态视觉定位模型")

def run_step(page, instruction: str) -> None:
    width = page.viewport_size["width"]
    height = page.viewport_size["height"]
    img = screenshot_b64(page)
    nx, ny = locate_target(img, instruction)      # 屏幕理解
    x, y = denormalize(nx, ny, width, height)      # 分辨率适配
    page.mouse.click(x, y)                         # 执行动作
    page.wait_for_timeout(500)                     # 等待观察新状态

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page(viewport={"width": 1440, "height": 900})
    page.goto("https://example.com")
    run_step(page, "点击页面上的『登录』按钮")
    browser.close()
```

这段代码的关键在于 `denormalize`：模型只负责输出语义化的归一化坐标，像素换算集中在适配层。当视口从 1440 宽变成 2560 宽，模型输出无需改变。

更进一步，可以优先从 a11y tree 取元素，避免视觉误判：

```python
def click_by_role(page, role: str, name: str) -> bool:
    """优先用语义化定位，成功返回 True，失败交给视觉回退。"""
    locator = page.get_by_role(role, name=name)
    if locator.count() == 1:
        locator.click()
        return True
    return False

# 策略：a11y 优先，视觉兜底
if not click_by_role(page, "button", "登录"):
    run_step(page, "点击页面上的『登录』按钮")
```

## 循环检测与工程实践建议

长任务最容易出的故障是「复读」：Agent 对着同一状态反复点同一个位置。一个简单的防御是记录最近若干步的动作指纹，出现高频重复就熔断。

```python
from collections import Counter

class LoopDetector:
    def __init__(self, window: int = 8, threshold: int = 4):
        self.window = window
        self.threshold = threshold
        self.history: list[str] = []

    def check(self, action_fp: str) -> bool:
        """返回 True 表示检测到循环，应中止任务。"""
        self.history.append(action_fp)
        self.history = self.history[-self.window:]
        return Counter(self.history).most_common(1)[0][1] >= self.threshold

# 动作指纹由「动作类型 + 归一化坐标 + 页面标题」组成
fp = f"click:{nx}:{ny}:{page.title()}"
if detector.check(fp):
    raise RuntimeError("检测到重复动作循环，任务中止")
```

工程上还有几条来自实战的建议。第一，**优先用 a11y tree 而非纯视觉**：语义信息稳定、不受缩放影响、定位成本更低，视觉只作回退。第二，**设置操作白名单**：限制可点击的元素类型与可访问的域名，禁止触碰删除、支付、提交表单等高风险控件，把爆炸半径压到最小。第三，**人在环审批**：涉及不可逆动作时暂停，把当前截图和拟执行动作推给人工确认。第四，**保留完整轨迹**：每一步的截图、动作、模型输出都要落盘，配合轨迹回放工具，才能在事故后定位是哪一步偏航。第五，**每步都设超时与状态校验**，动作执行后确认页面确实发生变化，否则按失败处理而非盲目推进。

## 小结

Computer Use Agent 的难度不在「让模型点中一次」，而在「让系统稳定点中一百次」。屏幕理解上，a11y tree 与视觉 grounding 应互补而非二选一；动作空间上，用归一化坐标和语义化动作屏蔽环境差异；容错上，循环检测、白名单、人在环和轨迹回放缺一不可。把这些地基打牢，模型能力的提升才能真正转化为可用的生产力。GUI Agent 的下一个竞争点，很可能不在于谁的模型更聪明，而在于谁的工程更稳。
