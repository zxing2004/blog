---
title: "用 Hugo 搭建一个零成本的技术博客"
date: 2026-01-15
draft: false
description: "从零开始，用 Hugo + GitHub Pages 搭建一个完全免费、自动部署的个人技术博客。"
tags: ["Hugo", "GitHub Pages", "静态站点"]
categories: ["工具与效率"]
---

## 为什么选择 Hugo

在众多静态站点生成器中，Hugo 有几个显著优势：

1. **极快的构建速度**——上千篇文章也能在秒级完成构建
2. **单一二进制文件**——无运行时依赖，跨平台开箱即用
3. **生态成熟**——丰富的主题与完善的中英文文档

更重要的是，配合 GitHub Pages，整套方案的成本为零，且无需维护服务器。

## 工作流程

整个博客的运作可以分为三个环节：

```bash
# 1. 本地写文章
hugo new posts/my-new-post.md

# 2. 本地预览（启动开发服务器）
hugo server -D

# 3. 提交推送，GitHub Actions 自动构建部署
git add . && git commit -m "new post" && git push
```

推送之后，GitHub Actions 会自动执行 `hugo --minify`，并将生成的 `public/` 目录部署到 GitHub Pages。

## 目录结构

一个典型的 Hugo 项目结构如下：

```
blog/
├── content/posts/     # 你的文章（Markdown）
├── layouts/           # 页面模板
├── static/            # 静态资源（CSS / JS / 图片）
├── archetypes/        # 新建文章的模板
└── hugo.toml          # 站点配置
```

## 小结

Hugo + GitHub Pages 是技术写作的高性价比组合：专注内容本身，把构建与部署交给自动化。接下来的文章会介绍如何定制主题和实现代码高亮。

> 本博客的源码已开源在 GitHub，欢迎 Star 与 Fork。
