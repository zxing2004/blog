---
title: "让 AI 写出能跑的前端：上下文、约束与可验证性"
slug: "ai-frontend-code-generation"
date: 2026-07-24
draft: false
description: "模型的前端编码能力再强，缺少上下文与约束也只会产出「看起来对」的页面。本文用上下文、约束、可验证性三要素，搭一套让 AI 稳定产出可用前端代码的工程闭环。"
tags: ["AI编程", "前端", "Prompt", "可验证性"]
categories: ["前端与体验"]
aliases:
  - "/posts/让-ai-写出能跑的前端上下文约束与可验证性/"
---

## 热点引入：前端代码竞技场登顶之后

2026 年 7 月，Kimi K3 在全球前端代码竞技场的榜单上登顶，同月 WAIC 召开，AI 生成前端代码的能力再次成为热议焦点。各类演示里，一句自然语言就能变出漂亮的落地页，似乎"前端躺平"时代已经到来。

竞技场的分数要打个折看。这类榜单靠人投票或模型互评产生，排名会随新模型入场、投票人群变化而浮动，同一模型在不同月份刷出的名次并不稳定（具体分数以榜单官方页面为准，本文不引用）。更重要的是，"从零生成一个漂亮落地页"和"在十万行的既有代码库里改对一处逻辑"，难度不在一个量级，前者恰恰是演示最喜欢挑的场景。

但一线团队的真实体验往往没那么丝滑：demo 里惊艳，落到自己的项目里就变成"样式对不上、组件用错、类型报错、跑不起来"。问题不在模型能力，而在**我们把一个需要工程约束的任务，当成了一次性问答**。模型的产出质量，高度依赖你给了多少上下文、设了多硬的约束、以及有没有一个能自动判定"能不能跑"的闭环。

本文把这件事拆成三要素——充分上下文、明确约束、可验证闭环，并给出可落地的模板与配置。

## 技术原理：决定产出质量的三个要素

**上下文（Context）**：模型不知道你的技术栈、设计系统、既有组件和代码风格。上下文越具体，它越不需要"猜"。一份好的上下文应包含技术栈版本、目录结构、设计 token、可复用组件清单、以及一两个"正确代码示例"作为风格锚点。

**约束（Constraint）**：上下文告诉模型"有什么"，约束告诉它"必须怎样、不许怎样"。约束要写成可判定的规则，例如"所有颜色只能用 CSS 变量，禁止硬编码 hex""按钮必须复用 `<Button>` 组件""所有交互元素需满足 keyboard 可达"。模糊的约束等于没有约束。

**可验证性（Verifiability）**：这是最容易被忽视、却最关键的一环。让模型写代码，必须有机器能自动判定对错的回路：type-check 过不过、单测过不过、页面截图和基线差多少。没有验证闭环，你只能靠肉眼 review，规模化根本无从谈起。

三者关系可以这样理解：**上下文与约束提升"一次生成"的命中率，可验证性保证"错了能被发现并修正"**。缺了前者靠反复重试，缺了后者错误会悄悄溜进代码库。

## 实操一：一份高质量 prompt 模板

下面是一份可直接复用的模板。关键是结构化——把技术栈、token、组件规范、可访问性分开陈述，并在末尾要求模型"先复述约束再动手"。

```text
# 角色
你是资深前端工程师，为本项目实现一个「订单列表页」。

# 技术栈（严格遵守版本）
- React 18 + TypeScript 5 + Vite 5
- 样式仅使用 CSS Modules + 项目 design token，禁止引入新的 UI 库
- 数据请求使用项目内的 useQuery 封装，禁止直接 fetch

# 设计 Token（只能引用，禁止硬编码颜色/间距）
- 颜色：var(--color-bg), var(--color-text), var(--color-brand)
- 间距：var(--space-1/2/3/4)
- 圆角：var(--radius-md)

# 组件规范
- 按钮必须复用 components/Button，props 见其 .d.ts
- 列表使用 components/Table，空态使用 components/Empty
- 所有组件从 @/components 统一导入

# 可访问性要求
- 所有交互元素可键盘操作，带 aria-label
- 表格使用 <th scope="col">，对比度符合 WCAG AA

# 交付要求
1. 先复述你在本题中必须遵守的约束（3-5 条）
2. 再输出完整文件代码，文件间用注释标明路径
3. 不要输出与任务无关的解释
```

"先复述约束"这一步看似多余，实则能显著降低模型忽略规则的概率——它被迫把约束纳入生成上下文。

## 实操二：用 design token 与组件契约锁住输出

把规范写成机器可读的文件，比写在 prompt 里更可靠。设计 token 用 CSS 变量定义，是模型无法"发明颜色"的物理边界：

```css
/* tokens.css —— 唯一合法的样式取值来源 */
:root {
  --color-bg: #ffffff;
  --color-text: #1f2937;
  --color-brand: #2563eb;
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --radius-md: 8px;
}
```

组件契用 TypeScript 接口表达，让模型清楚"能用什么、参数长什么样"：

```typescript
// components/Button/types.ts
export interface ButtonProps {
  /** 视觉层级 */
  variant: 'primary' | 'secondary' | 'ghost';
  /** 尺寸 */
  size?: 'sm' | 'md' | 'lg';
  /** 禁用状态 */
  disabled?: boolean;
  /** 无障碍标签，图标按钮必填 */
  'aria-label'?: string;
  onClick?: () => void;
  children: React.ReactNode;
}
```

把这些文件作为上下文喂给模型，它的输出就被夹在了一套确定的"围栏"里，而不是自由发挥。

## 实操三：可验证性闭环

生成之后，让机器先过一遍。下面是一个最小验收流水线：type-check、单测、Playwright 截图比对。

```json
// package.json (scripts 节选)
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "e2e": "playwright test",
    "verify": "npm run typecheck && npm run test && npm run e2e"
  }
}
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  expect: { toHaveScreenshot: { maxDiffPixelRatio: 0.01 } },
  use: { baseURL: 'http://localhost:5173', ...devices['Desktop Chrome'] },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: true,
  },
});
```

```typescript
// e2e/order-list.spec.ts
import { test, expect } from '@playwright/test';

test('订单列表页视觉回归', async ({ page }) => {
  await page.goto('/orders');
  await expect(page.getByRole('heading', { name: '订单列表' })).toBeVisible();
  // 与基线截图比对，差异超阈值即失败
  await expect(page).toHaveScreenshot('order-list.png');
});
```

`npm run verify` 一把跑完：类型不过、单测挂了、截图差异过大，任何一环失败都意味着这次生成不合格，必须回到模型侧修正。截图比对尤其重要——它把"看起来对不对"这种主观判断，变成了可自动执行的像素断言。

## 多轮迭代中的上下文一致性

生成很少一次到位，多轮修正是常态。要避免"改着改着风格跑偏"，关键是**维护一份不变的规范源**：每一轮都把同一份 token、组件契约和约束文件随请求一起带上，而不是依赖对话历史。对话历史会膨胀、会被截断，而文件是稳定的锚点。

另一个技巧是**小步提交 + 快照锁定**：每通过一次 `verify` 就提交一次，并用快照测试固定当前 UI 状态。后续修改若无意中改变了不该变的视觉，快照测试会立即报警。

## 工程实践建议与踩坑

- **把规范写进约束文件**。项目根目录的 `CLAUDE.md` / `AGENTS.md` 这类文件，会被编码助手自动读取，是放置技术栈、目录约定、代码风格的最佳位置——一次维护，处处生效。
- **给模型"正确示例"胜过十条禁令**。一个符合规范的组件样例，比反复强调"不要硬编码颜色"更有效。
- **验证闭环必须能自动判定**。任何需要人眼判断的验收环节都会成为规模化瓶颈；尽量把标准转化成 type-check、断言、截图 diff 这类机器可判定信号。
- **警惕依赖幻觉**。模型可能 import 一个不存在的包；type-check 和构建会暴露这类问题，务必纳入 `verify`。
- **可访问性要写进约束而非事后补救**。aria、keyboard、对比度这些要求，生成时一并提出成本最低，事后逐条补齐代价高昂。

## 小结

让 AI 写出"能跑"的前端，靠的不是更强的模型或更玄的提示词，而是三件朴素的工程事：**给足上下文**让它不必猜，**设硬约束**让它不能乱来，**建验证闭环**让错误无处遁形。模型能力是上限，而工程约束决定了你能不能稳定接近这个上限。把规范沉淀成文件、把验收变成脚本，AI 才会从"演示惊艳"真正走向"生产可用"。
