---
title: "Git 实用工作流：从提交规范到分支管理"
date: 2026-02-20
draft: false
description: "整理一套个人与团队都适用的 Git 工作流，涵盖提交信息规范、分支策略和常用技巧。"
tags: ["Git", "工程实践", "协作"]
categories: ["工具与效率"]
---

## 提交信息规范

清晰的提交信息是团队协作的基础。推荐采用 Conventional Commits 规范：

```
<类型>[可选作用域]: <描述>

[可选正文]

[可选脚注]
```

常见类型：

| 类型 | 含义 |
|------|------|
| `feat` | 新增功能 |
| `fix` | 修复缺陷 |
| `docs` | 文档变更 |
| `refactor` | 代码重构 |
| `chore` | 构建或辅助工具变更 |

## 分支管理

一个轻量的分支策略通常包含三类分支：

- `main` —— 稳定可发布的主分支
- `develop` —— 日常开发集成分支
- `feature/*` —— 功能开发分支，完成后合并回 `develop`

## 常用技巧

### 撤销上一次提交但保留改动

```bash
git reset --soft HEAD~1
```

### 修改最近一次提交信息

```bash
git commit --amend -m "new message"
```

### 交互式变整，合并多个提交

```bash
git rebase -i HEAD~3
```

### 找回误删的提交

```bash
# 查看所有分支的历史操作记录
git reflog

# 定位后创建新分支恢复
git branch recover-branch <commit-hash>
```

## 结语

好的 Git 习惯能显著降低协作成本。核心原则是：**小步提交、信息清晰、主线始终可用**。
