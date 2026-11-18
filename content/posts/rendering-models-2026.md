---
title: "2026 前端选型的底层逻辑：五种渲染模型对比"
date: 2026-10-23
draft: false
description: "2026 年前端选型的变量已从框架之争变为渲染模型之争。系统对比虚拟 DOM、Signals、Resumability、Islands 与 HTML-over-the-wire 五种模型，并给出按项目类型的决策建议。"
tags: ["前端架构", "渲染模型", "React", "Vue"]
categories: ["前端与体验"]
---

## 热点引入：框架之争，正在被渲染模型之争取代

2026 年 10 月，前端社区围绕「渲染模型取代框架之争」展开广泛讨论。React 19 带来 Compiler、Vue 推出 Vapor Mode、Svelte 5 的 Runes 落地、Qwik 2、Astro 5、HTMX 2 各自沿着不同路线演进。表面看是版本更迭，底层却是同一个转向：大家越来越意识到，真正决定应用性能与开发体验的，不是「你用 React 还是 Vue」，而是「你采用哪种渲染模型」。

过去几年，选型讨论长期停留在框架 API 与生态的对比上。但当这些框架在能力上趋于同质，差异反而集中到了更本质的地方——状态变化时，系统如何把变化映射到 DOM？首屏 HTML 之外，浏览器需要下载和执行多少 JS？交互发生时，更新是整树协调还是精确命中节点？这些问题的不同答案，构成了五种截然不同的渲染模型。

本文不站队，而是把五种模型放在同一套坐标系里拆解：原理是什么、适合什么场景、代价在哪里。最后结合平台原生能力的变化，给出「按项目类型选模型」的决策框架。

## 技术原理：五种渲染模型逐个讲透

**① 虚拟 DOM + 协调（React 19 + React Compiler）。** 状态变化后重新执行组件函数，生成一棵新的虚拟 DOM 树，与旧树 diff 出最小变更，再打补丁到真实 DOM。React Compiler 的职责是自动插入记忆化，减少不必要的重渲染，把过去靠开发者手写 `memo`/`useMemo` 的工作交给编译期。适用于交互复杂、组件树庞大的应用；代价是首屏要下载并运行 hydration 运行时，JS 体积偏大，且理解「何时重渲染」仍有心智成本。

**② Signals 细粒度响应式（Vue Vapor Mode / Svelte 5 Runes / Solid）。** 把状态包装成 signal，读取时自动收集依赖，变更时精确通知订阅者直接更新对应 DOM 节点，**跳过整棵组件树的 diff**。它把更新的粒度从「组件」下沉到「表达式」。适合高频交互、追求轻量运行时的场景；代价是生态仍在成长，调试时没有虚拟 DOM 快照可看，SSR 与 hydration 的成熟度各家不一。

**③ Resumability 可恢复性（Qwik 2）。** 服务端渲染后，把应用状态与事件监听器一并序列化进 HTML，浏览器端几乎不执行 hydration，只在用户真正交互时按需「恢复」对应片段。它追求的是接近零的 JS 启动成本。适合电商、内容站等对首屏与交互延迟极敏感的产品；代价是心智模型独特——开发者必须遵守序列化边界，第三方库兼容性也更棘手。

**④ Islands 岛屿架构（Astro 5）。** 默认输出纯静态 HTML，只在需要交互的区域插入彼此独立的「岛屿」，各自单独 hydration，甚至可以在同一页混合不同框架。适合内容为主、局部交互的场景（博客、文档、营销页）；代价是岛屿之间共享状态成本高，不适合整体高度交互的应用。

**⑤ HTML-over-the-wire（HTMX 2）。** 交互由服务端处理，返回的不是 JSON 而是 HTML 片段，浏览器直接替换目标 DOM。它把客户端逻辑大幅搬回服务端，弱化前端 JS。适合表单密集的内部工具与服务端渲染体系（Rails、Django）；代价是客户端状态管理能力弱，复杂交互体验受限，且体验受网络延迟影响明显。

## 实操：从 Signals 到各模型的最小形态

理解差异最好的办法是看代码。先看 Signals 的核心机制——依赖收集与精确通知，几十行就能还原其精髓。

```js
let activeEffect = null;

function signal(initial) {
  let value = initial;
  const subs = new Set();
  return {
    get() {
      if (activeEffect) subs.add(activeEffect);  // 读取时收集依赖
      return value;
    },
    set(next) {
      if (Object.is(value, next)) return;
      value = next;
      for (const fn of [...subs]) fn();          // 写入时精确通知
    },
  };
}

function effect(fn) {
  const run = () => { activeEffect = run; fn(); activeEffect = null; };
  run();
}
function computed(fn) {
  const out = signal(undefined);
  effect(() => out.set(fn()));                   // 依赖变化时重算
  return out;
}

// 使用：只更新真正依赖 total 的节点，无需虚拟 DOM diff
const price = signal(10), count = signal(2);
const total = computed(() => price.get() * count.get());
effect(() => { document.querySelector("#total").textContent = total.get(); });
count.set(3);   // 仅 #total 被更新
```

再看不同模型下组件写法的差异。Signals 系（Solid）直接声明响应式原语；Resumability 系（Qwik 2）用 `$` 后缀标记可序列化边界——这是它按需恢复的关键约束。

```jsx
// ② Solid：细粒度响应式，编译后无虚拟 DOM
function Counter() {
  const [count, setCount] = createSignal(0);
  return <button onClick={() => setCount(count() + 1)}>{count()}</button>;
}

// ③ Qwik 2：component$ / onClick$ 标记可恢复边界
export const Counter = component$(() => {
  const count = useSignal(0);
  return <button onClick$={() => count.value++}>{count.value}</button>;
});
```

而 Islands 与 HTML-over-the-wire 干脆把交互诉求写在 HTML 层：Astro 5 用 `client:load` 决定哪个组件激活，HTMX 2 用 `hx-post` 把请求结果直接注入目标节点——两者都在努力减少「必须执行的客户端 JS」。

## 工程实践：五模型对比与决策建议

把五种模型放在同一套维度下对比，选型会清晰很多。

| 渲染模型 | 首屏成本 | 交互复杂度上限 | SEO | 生态成熟度 | 心智负担 |
| --- | --- | --- | --- | --- | --- |
| 虚拟 DOM + 协调 | 中（hydration） | 高 | 好（配 SSR） | 最成熟 | 中高 |
| Signals | 低 | 高 | 好 | 成长中 | 中 |
| Resumability | 极低 | 高 | 好 | 较小 | 高 |
| Islands | 极低 | 中低 | 极好 | 较好 | 低 |
| HTML-over-the-wire | 低 | 中低 | 好 | 中等 | 低 |

**按项目类型选模型：**

- **内容 / 文档 / 营销站**：首选 Islands（Astro 5），默认零 JS、SEO 极佳，局部交互用岛屿补齐。
- **表单密集的内部工具**：首选 HTML-over-the-wire（HTMX 2），交互逻辑留在服务端，迭代快、客户端包极小。
- **复杂 SPA（协作工具、编辑器、数据看板）**：在虚拟 DOM（React 19，生态与人才储备最强）与 Signals（Solid / Svelte 5，运行时更轻）之间二选一，取决于团队规模与对细粒度性能的诉求。
- **首屏敏感、流量巨大的电商 / 内容产品**：可评估 Resumability（Qwik 2），但要把独特心智模型与生态兼容成本计入账。

**平台原生能力正在改变权重。** View Transitions API 把跨状态过渡动画下沉到浏览器，Navigation API 把客户端路由接管能力交给原生。这意味着过去需要重型框架才能实现的「流畅转场」与「无刷新导航」，如今用原生 API 配合 Islands 就能做到。选型时不妨先问：这个能力还需要框架运行时吗？原生能力越强，越削弱「为一个动画引入整套框架」的理由。

```js
// View Transitions：原生跨状态过渡
function switchTheme(fn) {
  if (!document.startViewTransition) return fn();
  document.startViewTransition(() => fn());
}

// Navigation API：原生拦截导航、接管路由
navigation.addEventListener("navigate", (event) => {
  if (!event.canIntercept || event.hashChange) return;
  event.intercept({ async handler() {
    const html = await fetch(event.destination.url).then(r => r.text());
    document.querySelector("main").innerHTML = html;
  }});
});
```

几个常见踩坑：SSR 与水合不一致（hydration mismatch）会导致首屏闪烁或事件失效；Islands 之间共享状态需要显式设计，别指望全局状态凭空打通；Resumability 的序列化边界一旦被忽略，调试会非常痛苦；HTMX 的体验受网络往返影响，交互密集场景要谨慎。

## 小结

2026 年的前端选型，先定渲染模型，再选具体框架，顺序不应颠倒。五种模型没有绝对优劣：虚拟 DOM 赢在生态，Signals 赢在细粒度，Resumability 赢在启动成本，Islands 赢在内容场景的极简，HTML-over-the-wire 赢在服务端体系的开发效率。把项目的交互复杂度、首屏要求、SEO 与团队心智放在一起权衡，再叠加平台原生能力带来的新变量，才能做出经得起时间检验的架构决策。
