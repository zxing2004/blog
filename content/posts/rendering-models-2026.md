---
title: "2026 前端选型的底层逻辑：五种渲染模型对比"
date: 2026-10-23
draft: false
description: "2026 年前端选型的变量已从框架之争变为渲染模型之争。但虚拟 DOM、Signals、Resumability、Islands 与 HTML-over-the-wire 并不在同一层，混着比会得出错误结论。本文先把它们按交付层与更新层归位，再逐层对比并给出决策建议。"
tags: ["前端架构", "渲染模型", "React", "Vue"]
categories: ["前端与体验"]
---

## 热点引入：框架之争，正在被渲染模型之争取代

2026 年 10 月，前端社区围绕「渲染模型取代框架之争」展开广泛讨论。React 19 生态里 Compiler 1.0 正式可用、Vue 推出 Vapor Mode、Svelte 5 的 Runes 落地、Qwik 2、Astro 5、HTMX 2 各自沿着不同路线演进。表面看是版本更迭，底层却是同一个转向：大家越来越意识到，真正决定应用性能与开发体验的，不是「你用 React 还是 Vue」，而是「你采用哪种渲染模型」。

过去几年，选型讨论长期停留在框架 API 与生态的对比上。但当这些框架在能力上趋于同质，差异反而集中到了更本质的地方——状态变化时，系统如何把变化映射到 DOM？首屏 HTML 之外，浏览器需要下载和执行多少 JS？交互发生时，更新是整树协调还是精确命中节点？这些问题的不同答案，构成了几种截然不同的渲染模型。

需要先纠正一个常见误解：所谓「五种模型」并不是五支可以并排打分的队伍。它们回答的是两个不同层面的问题——**交付层**（HTML 送到浏览器之后，还要下载多少 JS、什么时候把它「激活」）和**更新层**（状态一变，变更如何落到真实 DOM 上）。虚拟 DOM 与 Signals 主要回答更新层，Resumability、Islands、HTML-over-the-wire 主要回答交付层。把一个交付层方案和两个更新层方案摆成三选一，是这类对比文章最常见的逻辑漏洞。

所以本文的拆法不是让五者同台竞技，而是先按层归位、再逐层比较：每种模型的原理是什么、它到底优化了哪一层、代价落在哪里。最后再回答那个真正要落地的问题——给定一类项目，该在哪一层下注。

## 技术原理：五种渲染模型逐个讲透

**① 虚拟 DOM + 协调（React 19 + React Compiler）。** 状态变化后重新执行组件函数，生成一棵新的虚拟 DOM 树，与旧树 diff 出最小变更，再打补丁到真实 DOM。这里要点明一个常见的时间线混淆：Compiler 不是 React 19 的运行时特性，它是一个独立的编译期工具（1.0 于 2025 年 10 月发布，需自己接入 Babel/Vite 插件与运行时包），作用是在编译期自动插入记忆化，把过去靠手写 `memo`/`useMemo` 的工作接管过去。适用于交互复杂、组件树庞大的应用；代价是首屏要下载并运行 hydration 运行时，JS 体积偏大，且理解「何时重渲染」仍有心智成本。

**② Signals 细粒度响应式（Vue Vapor Mode / Svelte 5 Runes / Solid）。** 把状态包装成 signal，读取时自动收集依赖，变更时精确通知订阅者直接更新对应 DOM 节点，**跳过整棵组件树的 diff**。它把更新的粒度从「组件」下沉到「表达式」。适合高频交互、追求轻量运行时的场景；代价是生态仍在成长，调试时没有虚拟 DOM 快照可看，SSR 与 hydration 的成熟度各家不一。

**③ Resumability 可恢复性（Qwik 2）。** 服务端渲染后，把应用状态与事件监听器一并序列化进 HTML，浏览器端几乎不执行 hydration，只在用户真正交互时按需「恢复」对应片段。它追求的是接近零的 JS 启动成本。适合电商、内容站等对首屏与交互延迟极敏感的产品；代价是心智模型独特——开发者必须遵守序列化边界，第三方库兼容性也更棘手。

**④ Islands 岛屿架构（Astro 5）。** 默认输出纯静态 HTML，只在需要交互的区域插入彼此独立的「岛屿」，各自单独 hydration。关键一点它和前三者不是同类：Islands 是一种**组合策略**，岛内部用什么模型由你决定——放一个 React 组件，那颗岛里跑的就是虚拟 DOM 与协调；放 Svelte 组件，跑的就是 signals；甚至可以同一页混用不同框架。所以「选 Islands」和「选 Signals」从来不是互斥的，前者决定交付层，后者决定岛内的更新层。适合内容为主、局部交互的场景（博客、文档、营销页）；代价是岛屿之间共享状态成本高，不适合整体高度交互的应用。

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

把五种模型放进同一张表时，必须先标明它们各自作用于哪一层，否则「首屏成本」这种指标会被误读成横向可比。

| 渲染模型 | 主要作用于 | 首屏成本 | 交互复杂度上限 | SEO | 生态成熟度 | 心智负担 |
| --- | --- | --- | --- | --- | --- | --- |
| 虚拟 DOM + 协调 | 更新层（附带 hydration 交付成本） | 中 | 高 | 好（配 SSR） | 最成熟 | 中高 |
| Signals | 更新层 | 低 | 高 | 好（配 SSR） | 成长中 | 中 |
| Resumability | 交付层 | 极低 | 高 | 好 | 较小 | 高 |
| Islands | 交付层（组合策略，可承载前两者） | 极低 | 中低 | 极好 | 较好 | 低 |
| HTML-over-the-wire | 交付层 | 低 | 中低 | 好 | 中等 | 低 |

看表时容易踩两个坑。一是把 Islands 的「交互复杂度上限 中低」读成"能力弱"——它说的是"整页高度联动"的场景不适合，而不是岛内不能复杂；一颗岛里放 React，岛内的复杂度上限就是 React 的上限。二是把 Resumability 的「首屏成本 极低」当成普遍优势——它只对首屏冷启动成立，一旦用户把整页交互点遍，按需加载的 JS 总量可能反超，取舍要看真实交互分布。

**按项目类型选模型：**

- **内容 / 文档 / 营销站**：首选 Islands（Astro 5），默认零 JS、SEO 极佳，局部交互用岛屿补齐。
- **表单密集的内部工具**：首选 HTML-over-the-wire（HTMX 2），交互逻辑留在服务端，迭代快、客户端包极小。
- **复杂 SPA（协作工具、编辑器、数据看板）**：在虚拟 DOM（React 19，生态与人才储备最强）与 Signals（Solid / Svelte 5，运行时更轻）之间二选一，取决于团队规模与对细粒度性能的诉求。
- **首屏敏感、流量巨大的电商 / 内容产品**：可评估 Resumability（Qwik 2），但要把独特心智模型与生态兼容成本计入账。

**平台原生能力正在改变权重。** View Transitions API 把跨状态过渡动画下沉到浏览器，Navigation API 把客户端路由接管能力交给原生。这意味着过去需要重型框架才能实现的「流畅转场」与「无刷新导航」，如今用原生 API 配合 Islands 就能做到。选型时不妨先问：这个能力还需要框架运行时吗？原生能力越强，越削弱「为一个动画引入整套框架」的理由。

```js
// View Transitions：原生跨状态过渡
function switchTheme(fn) {
  if (!document.startViewTransition) return fn();   // 不支持就同步执行，别让功能依赖它
  document.startViewTransition(() => fn());
}

// Navigation API：原生拦截导航、接管路由
navigation.addEventListener("navigate", (event) => {
  // 能接管、非锚点跳转、非下载、同源——四条都满足才拦
  if (!event.canIntercept || event.hashChange || event.downloadRequest) return;
  const url = new URL(event.destination.url);
  if (url.origin !== location.origin) return;

  event.intercept({
    async handler() {
      const html = await fetch(url).then((r) => r.text());
      const next = new DOMParser().parseFromString(html, "text/html");
      const incoming = next.querySelector("main");
      if (!incoming) {
        location.href = url.href;        // 拿不到片段，退回整页跳转
        return;
      }
      document.querySelector("main").replaceWith(incoming);
      document.title = next.title;
    },
  });
});
```

这段代码里两个细节值得注意。第一，`fetch` 回来的是**整页 HTML**，直接 `innerHTML = html` 会把整个 `<html>` 文档塞进 `<main>` 里，浏览器虽然会容错解析，但样式表和脚本会被重复插入——所以必须先用 `DOMParser` 解析、只取目标片段。第二，`replaceWith` 会连同旧节点上绑定的事件监听一起丢掉，这正是「原生路由接管」的代价：页面骨架换了，行为得自己重新挂。这也是为什么原生方案更适合配合 Islands——只有少数区域需要重新绑定。

几个常见踩坑：SSR 与水合不一致（hydration mismatch）会导致首屏闪烁或事件失效；Islands 之间共享状态需要显式设计，别指望全局状态凭空打通；Resumability 的序列化边界一旦被忽略，调试会非常痛苦；HTMX 的体验受网络往返影响，交互密集场景要谨慎。

## 小结

2026 年的前端选型，先定渲染模型，再选具体框架，顺序不应颠倒。但「定渲染模型」要分两次问：**交付层**先回答"HTML 之外还要给浏览器多少 JS、什么时候激活"，**更新层**再回答"变更怎么落到 DOM"。前一问落在 Resumability、Islands、HTML-over-the-wire 之间，后一问落在虚拟 DOM 与 Signals 之间，两问的答案可以自由组合——Astro 岛屿里放 React，就是"Islands 交付 + 虚拟 DOM 更新"；同一份 Islands 换成 Svelte，交付层不变、更新层变轻。

单看某一层，各有各的赢法：虚拟 DOM 赢在生态与人才，Signals 赢在细粒度更新的运行时开销，Resumability 赢在冷启动，Islands 赢在内容站的默认零 JS，HTML-over-the-wire 赢在服务端体系的开发效率。把这些优势按层对齐，再把项目的交互复杂度、首屏要求、SEO 与团队心智摊上去权衡，最后叠加平台原生能力带来的新变量，才是经得起时间检验的架构决策。
