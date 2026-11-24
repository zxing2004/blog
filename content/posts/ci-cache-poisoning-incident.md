---
title: "一次 CI 缓存污染导致的诡异构建事故"
slug: "ci-cache-poisoning-incident"
date: 2026-05-21
draft: false
description: "构建产物偶发异常，本地怎么都复现不了，只有 CI 上某些任务出问题。复盘一次由缓存 key 设计粗糙引发的构建事故。"
tags: ["CI/CD", "GitLab CI", "缓存", "可重现构建"]
categories: ["运维部署"]
---

## 故障背景

5 月中旬的一个周四，前端同学发现一个怪现象：同一个 commit 的代码，在 CI 上跑出来的产物偶尔会带上一版旧的资源文件，部署后页面白屏。但同样的代码拉到本地构建，一切正常。更奇怪的是，不是每次构建都出问题——重跑一次流水线，往往又好了。

这种"本地复现不了、CI 偶发"的问题最耗精力。当时正值版本迭代密集期，每天几十次构建，偶发故障很难和某次具体改动对上号。我们一开始怀疑是构建节点环境不一致，检查了 Node 版本、环境变量、时区和依赖安装顺序，全都没有差异。直到把几次失败和成功的构建日志并排放在一起看，一条不起眼的差异浮了出来。

## 异常现象

失败的构建日志里，缓存恢复阶段无一例外都是 `Restoring cache` 成功，而重跑成功的那次显示 `WARNING: file does not exist`——缓存没命中。问题瞬间和缓存对上了号。

白屏的原因得从产物本身找。前端是 hash 命名 + `index.html` 引用的结构，如果 `index.html` 和资源文件不是同一次构建出来的，就会互相找不到。我们直接比对产物：

```bash
$ git rev-parse --short HEAD
9f8e7d6
$ cat dist/version.json
{"version": "2.3.1", "commit": "a1b2c3d", "builtAt": "2026-05-11T09:22:11Z"}
$ grep -oE 'assets/index-[a-z0-9]+\.js' dist/index.html
assets/index-4c81f2a9.js
$ ls dist/assets/ | head -5
index-1e77b0c4.js
index-4c81f2a9.js
vendor-2d91ab77.js
```

三处对不上：当前代码是 `9f8e7d6`，产物里记着 `a1b2c3d`——十天前的提交；`index.html` 引用的是 `index-4c81f2a9.js`，但同一次构建产出的主 chunk 是 `index-1e77b0c4.js`；而 `4c81f2a9` 这个文件压根不在 `dist/assets/` 里。浏览器拿着 `index.html` 去请求一个不存在的 chunk，直接白屏。

这不是"构建出错"，是**新旧两次构建的产物被混在了一起**。

## 排查思路

登到 runner 上看缓存目录。缓存按分支名存放：

```bash
$ du -sh /cache/app-web/* | sort -h | tail -3
 12M    /cache/app-web/feature-payment
298M    /cache/app-web/main
406M    /cache/app-web/release-2.3
$ ls /cache/app-web/main/
dist/  node_modules/
```

`dist/` 在缓存里。这就是答案的一半——构建产物被当成缓存恢复了。

再看流水线配置：

```yaml
# 问题配置
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - dist/
```

key 只有分支名，`main` 分支所有构建共用同一份缓存，永不过期；`paths` 里同时缓存了依赖和产物。流程就变成了：`dist/` 从十天前的某次构建里被恢复出来 → 构建工具按增量逻辑只重新生成了一部分 chunk → `index.html` 被新生成，但引用的是上一次构建留下的 manifest/旧 chunk 名 → 新旧混杂。

为了确认不是 node_modules 或环境的问题，我们做了一组对照：在同一个 commit 上，清空缓存跑一次（成功），保留缓存跑一次（失败），连跑三轮结果稳定复现。同时把 `dist/` 从缓存里临时摘掉再跑，问题消失。到这里可以确认：污染源是 `dist/`，跟依赖、Node 版本都无关。

## 处理过程

先清空 runner 上的相关缓存，再改 key 的设计：

```yaml
# 修正后：key 绑定 lockfile 哈希并按分支隔离，产出目录不再缓存
cache:
  key:
    prefix: ${CI_COMMIT_REF_SLUG}
    files:
      - package-lock.json
  paths:
    - node_modules/
    - .npm/
```

三处改动：`key` 加了 `files`，lockfile 一变 key 就变，旧缓存自动失效；保留 `prefix` 做分支隔离；`dist/` 从 `paths` 里删掉——构建产物每次必须从零生成。

改完还要验证"从零生成"这件事真的成立。我们在流水线末尾加了一步自检，连续构建两次比对产物哈希：

```bash
$ npm ci --prefer-offline --no-audit
$ npm run build
$ find dist -type f -name '*.js' -o -name '*.css' | sort | xargs sha256sum | sha256sum
7f3c9e21b8d4a0f6...  -
```

同一 commit 连跑两次哈希一致，才算通过。这条自检上线后又抓到两条流水线也在缓存 `dist/`，一并清掉。

## 根因分析

事故的根因是缓存 key 太粗，以及把产物当成了缓存。

原来的 key 只取分支名，粒度停留在"分支"这个层级，而真正决定构建结果的是代码内容。代码变了、key 没变，缓存照样命中，十几天前的 `dist/` 就被原样搬进新的工作目录。构建工具看到 `dist/` 里已经有东西，加上自己还有增量缓存，就只补了缺失的部分——于是 `index.html` 是新的，chunk 是旧的，引用关系断掉。

更深一层看，这是**把"加速手段"当成了"状态存储"**。缓存本该是可随时丢弃的派生数据，我们却让它承载了影响产出的关键状态。一旦内容失真，整个构建就失去了可重现性。

这里还有两个认知偏差。一是把 `dist/` 当缓存：缓存应该只放"输入型"的东西（依赖包、下载物），产物是构建的输出，缓存它等于把上一次的结果当成这一次的输入。二是过度信任"缓存命中率"这个指标：命中率越高看着越省时间，但如果命中的是错的内容，省下的每一秒都在给故障加速。

还有个具体的技术细节值得记下来：hash 命名的资源文件本身是为了缓存友好，但它假设"一次构建产出的所有文件属于同一个集合"。一旦新旧产物混在一起，这个假设就崩了——而且崩得很安静，构建日志全绿，`dist/` 也在，只有浏览器控制台会报 404。

## 复盘总结

事后我们整理了团队缓存使用的几条原则，写进 CI 模板，新项目直接继承。

第一，缓存 key 必须包含决定构建结果的内容哈希，最直接的就是 lockfile 哈希或源码树的 commit；不允许只用分支名或固定字符串。

第二，缓存只放依赖，不放产物。`dist/`、`build/`、`.next/`、`target/` 这类目录一律不进缓存，构建每次从零开始。

第三，加了两道校验：把源码 commit 和 lockfile 哈希写进产物元数据，部署前比对，不一致就拒绝发布；"同一 commit 连续构建两次，产物哈希必须一致"列为流水线验收项。

第四，定期清理长期未命中的旧缓存。`release-2.3` 那份 406M 的缓存挂了两个月没人用，本身就是噪音。

回想当时，最浪费时间的不是查不出问题，而是我们一直默认缓存是可信的。缓存是典型的"快但危险"的优化：它用正确性换速度，而正确性一旦出问题，往往以最隐蔽的方式出现——不是全挂，而是偶尔错一点。那些"本地好的、CI 偶发"的问题，值得优先怀疑缓存。
