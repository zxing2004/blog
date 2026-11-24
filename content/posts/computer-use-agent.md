---
title: "Computer Use Agent 的工程难点：屏幕理解与动作空间"
date: 2026-10-16
draft: false
description: "GUI Agent 从演示走向生产，难在屏幕理解、动作空间设计与容错可观测性。本文拆解 CUA 闭环，并用 Playwright 给出可运行的最小实现与循环检测方案。"
tags: ["Computer Use", "GUI Agent", "Agent", "自动化"]
categories: ["工具与效率"]
---

## 热点引入：从「看懂屏幕」到「动手操作」

2026 年 9 月，上海创智学院开源了 Nex 系列模型，主打「所见即行动」的电脑操控能力（Computer Use Agent，CUA），在开源社区引来一波实测。11 月的后续版本进一步降低了长任务中的复读率——也就是 Agent 反复执行同一无效动作、卡在原地的现象（各版本的具体指标以官方发布为准）。短短两个月，GUI Agent 从论文里的炫技演示，变成了开发者手里真想接进工作流的工具。

为什么现在值得认真讨论它的工程问题？因为「能点对一次」和「能稳定干完一百步」是两回事。前者是模型能力问题，后者是系统设计问题。屏幕分辨率会变、界面会改版、弹窗会突然出现、网络会抖动——真实桌面环境的不确定性，远比聊天框里的一问一答复杂。

本文不谈模型有多强，而是拆解一个 CUA 系统必须解决的三个工程难点：屏幕理解、动作空间设计、容错与可观测性，并给出可直接改造落地的最小实现。

## 技术原理：CUA 的闭环与三道难关

所有 Computer Use Agent 都跑在同一个闭环上：**截图 → 屏幕理解 → 规划 → 执行动作 → 观察新状态 → 继续**。每一步的观察都来自屏幕，每一步的动作都落在屏幕。这个「看—想—动」循环看似简单，工程上却处处是坑。

**第一关是屏幕理解。** 模型需要把一张像素图翻译成「这里有个按钮、那里是个输入框」。纯视觉方案依赖视觉 grounding：模型输出目标元素的坐标。但坐标会受分辨率、缩放、DPI 影响，一个在 1920×1080 上正确的像素点，换到 2560×1440 就偏了。更稳妥的做法是利用可访问性树（a11y tree）：浏览器和操作系统会暴露界面的语义结构——角色、名称、层级、区域。

这里要澄清一个常见误记：**a11y tree 本身并不携带边界框**。它给的是 role、name、state 和层级关系，规范里没有几何信息。真正拿到坐标要靠另外两步——在浏览器里是用 CDP 的 `DOM.getBoxModel`、或 Playwright 的 `locator.bounding_box()`（底层都是 `getBoundingClientRect`）；在桌面端则要借助平台的无障碍 API 配合窗口坐标换算。所以「a11y tree 天然给出语义和边界框」这个说法只对了一半：语义是天然的，坐标得补。工程上的常见做法是先用 a11y tree 锁定候选元素（拿语义和唯一性），再补一次几何查询拿到矩形中心，最后用视觉做校验和回退。

**第二关是动作空间设计。** 动作不是越多越好，关键是坐标系与粒度。最常用的是 `click / type / scroll / key` 四件套：点击需要 (x, y) 坐标，输入需要文本与焦点位置，滚动需要方向和幅度，按键需要组合键描述。坐标系必须统一——建议对外暴露 0–1000 的归一化坐标，由适配层按当前视口换算成真实像素，这样同一份模型输出能跨分辨率复用。粒度上，应尽量使用语义化动作（如「点击名为『提交』的按钮」）而非裸坐标，把像素细节留在最底层。

**第三关是容错与可观测性。** GUI 环境充满意外：动作没生效、页面没加载、误触弹窗。系统必须能检测失败、识别循环，并保留完整轨迹供回放。没有轨迹回放，线上出问题时你根本无从复现——因为用户的屏幕状态你无法重来。

## 实操：用 Playwright 实现最小 GUI Agent 循环

下面给出一个可运行的最小骨架。核心思路：截图后调用视觉模型返回目标元素的归一化坐标，再换算成真实像素执行点击。模型调用部分用伪代码接口表示，便于替换成任意多模态模型。

```python
import base64
import hashlib
from playwright.sync_api import sync_playwright

# 对外统一用 0~1000 的归一化坐标系，屏蔽分辨率差异
def denormalize(nx: int, ny: int, width: int, height: int) -> tuple[int, int]:
    return int(nx / 1000 * width), int(ny / 1000 * height)

def screenshot_b64(page) -> str:
    return base64.b64encode(page.screenshot()).decode("utf-8")

def state_hash(page) -> str:
    """用截图哈希代表「当前画面」，作为状态指纹。"""
    return hashlib.sha1(page.screenshot()).hexdigest()[:12]

def wait_for_change(page, before: str, timeout_ms: int = 3000, step: int = 200) -> bool:
    """等画面真的变了，而不是盲等固定时长。返回是否观察到变化。"""
    waited = 0
    while waited < timeout_ms:
        page.wait_for_timeout(step)
        waited += step
        if state_hash(page) != before:
            return True
    return False

def locate_target(image_b64: str, instruction: str) -> tuple[int, int]:
    """调用视觉模型，返回目标元素的归一化坐标 (nx, ny)。
    实际实现中把 image_b64 与 instruction 发给多模态模型，
    要求其只输出 JSON: {"nx": int, "ny": int}，再做解析校验。
    """
    raise NotImplementedError("接入你的多模态视觉定位模型")

def run_step(page, instruction: str) -> bool:
    """执行一步。返回 False 表示动作没有引起画面变化，需按失败处理。"""
    width = page.viewport_size["width"]
    height = page.viewport_size["height"]
    before = state_hash(page)                      # 动作前的状态
    img = screenshot_b64(page)
    nx, ny = locate_target(img, instruction)       # 屏幕理解
    x, y = denormalize(nx, ny, width, height)      # 分辨率适配
    page.mouse.click(x, y)                         # 执行动作
    return wait_for_change(page, before)           # 观察新状态并校验

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page(viewport={"width": 1440, "height": 900})
    page.goto("https://example.com")
    if not run_step(page, "点击页面上的『登录』按钮"):
        print("动作未生效：画面没有变化，转人工或重试")
    browser.close()
```

这段代码有两个关键设计。一是 `denormalize`：模型只负责输出语义化的归一化坐标，像素换算集中在适配层，视口从 1440 宽变成 2560 宽时模型输出无需改变。二是 `wait_for_change`：`page.wait_for_timeout(500)` 这种盲等在生产里两头不讨好——页面快时白等，页面慢时不够；改成「等到画面真的变了或超时」既省时间又天然带了一层结果校验，动作没生效会被当场发现，而不是被后面的步骤带偏。

不过截图哈希有个已知缺陷：页面上的时钟、轮播图、骨架屏动画会让每一帧都不一样，哈希永远在变，于是「画面变了」恒为真，校验形同虚设。实践中要么改用更稳的状态表征（比如 a11y 快照的文本摘要、或 `main` 区域的 DOM 结构哈希），要么对截图做感知哈希并设一个容差阈值。

更进一步，可以优先从 a11y tree 取元素，避免视觉误判：

```python
def click_by_role(page, role: str, name: str) -> bool:
    """优先用语义化定位，成功返回 True，失败交给视觉回退。"""
    # exact=True 很重要：默认是子串匹配，"登录" 会同时命中 "登录/注册"
    locator = page.get_by_role(role, name=name, exact=True)
    if locator.count() == 1:
        locator.click()
        return True
    return False

# 策略：a11y 优先，视觉兜底
if not click_by_role(page, "button", "登录"):
    run_step(page, "点击页面上的『登录』按钮")
```

注意 `get_by_role` 的 `name` 参数默认是**不区分大小写的子串匹配**，不是精确匹配。页面上同时有「登录」和「登录/注册」两个按钮时，不加 `exact=True` 会命中两个，`count()` 返回 2，代码静默走到视觉回退——看着没报错，实际是语义定位白做了一次。这类"没失败但也没起作用"的降级，比直接报错更难查。

## 循环检测与工程实践建议

长任务最容易出的故障是「复读」：Agent 对着同一状态反复点同一个位置。一个简单的防御是记录最近若干步的动作指纹，出现高频重复就熔断。

```python
from collections import Counter

class LoopDetector:
    def __init__(self, window: int = 8, threshold: int = 3):
        self.window = window
        self.threshold = threshold
        self.action_history: list[str] = []
        self.state_history: list[str] = []

    def check(self, action_fp: str, state_fp: str) -> bool:
        """返回 True 表示检测到循环，应中止任务。

        - 动作 + 状态 都不变：原地空转
        - 只有少数几个状态来回出现：在两个画面之间震荡
        """
        self.action_history = (self.action_history + [f"{action_fp}@{state_fp}"])[-self.window:]
        self.state_history = (self.state_history + [state_fp])[-self.window:]

        repeat_action = Counter(self.action_history).most_common(1)[0][1] >= self.threshold
        # 画面重复出现说明在原地打转，哪怕动作每次不同
        repeat_state = Counter(self.state_history).most_common(1)[0][1] >= self.window // 2
        return repeat_action or repeat_state


detector = LoopDetector(window=8, threshold=3)

def viewport(page) -> tuple[int, int]:
    s = page.viewport_size
    return s["width"], s["height"]

def plan_next_action(page) -> tuple[int, int]:
    """占位：真实实现里是「截图 + 当前目标 → 下一步归一化坐标」。"""
    raise NotImplementedError

# 每一轮循环里，动作指纹由「动作类型 + 归一化坐标」组成，状态指纹来自截图
for step_no in range(100):
    before = state_hash(page)
    nx, ny = plan_next_action(page)          # 模型规划
    if detector.check(f"click:{nx}:{ny}", before):
        raise RuntimeError(f"第 {step_no} 步检测到循环，任务中止")
    page.mouse.click(*denormalize(nx, ny, *viewport(page)))
    wait_for_change(page, before)
```

这里有两处容易写歪。第一，**动作指纹必须带上状态指纹**。只用「动作类型 + 坐标」时，正常的重复操作会被误判——比如逐个点开列表里的条目然后返回，动作一模一样，但每次都在推进。反过来，只用动作也不行：Agent 在两个页面之间来回弹（登录页 → 报错页 → 登录页），每步动作都不同，纯动作指纹一次也抓不到，所以需要第二个以状态为单位的判据（`repeat_state`）。两道判据各自的误报率都低，合起来才够用。

第二，`window` 与 `threshold` 的取值本身是个权衡：窗口太小容易把正常的长表单填写判成循环，太大则等到烧掉几十步才熔断。我的经验值是 `window=8`、`threshold=3`——连续 8 步里有 3 次完全相同的「动作 + 画面」，基本可以认定卡住了。

工程上还有几条来自实战的建议。第一，**优先用 a11y tree 而非纯视觉**：语义信息稳定、不受缩放影响、定位成本更低，视觉只作回退。第二，**设置操作白名单**：限制可点击的元素类型与可访问的域名，禁止触碰删除、支付、提交表单等高风险控件，把爆炸半径压到最小。第三，**人在环审批**：涉及不可逆动作时暂停，把当前截图和拟执行动作推给人工确认。第四，**保留完整轨迹**：每一步的截图、动作、模型输出都要落盘，配合轨迹回放工具，才能在事故后定位是哪一步偏航。第五，**每步都设超时与状态校验**，动作执行后确认页面确实发生变化，否则按失败处理而非盲目推进。

## 小结

Computer Use Agent 的难度不在「让模型点中一次」，而在「让系统稳定点中一百次」。屏幕理解上，a11y tree 与视觉 grounding 应互补而非二选一；动作空间上，用归一化坐标和语义化动作屏蔽环境差异；容错上，循环检测、白名单、人在环和轨迹回放缺一不可。把这些地基打牢，模型能力的提升才能真正转化为可用的生产力。GUI Agent 的下一个竞争点，很可能不在于谁的模型更聪明，而在于谁的工程更稳。
