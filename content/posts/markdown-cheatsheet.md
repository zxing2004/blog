---
title: "Markdown 写作速查与进阶技巧"
date: 2026-04-05
draft: false
description: "一份面向技术写作的 Markdown 速查表，涵盖标题、代码块、表格、脚注等常用语法。"
tags: ["Markdown", "写作", "Hugo"]
categories: ["工具与效率"]
---

## 标题

使用 `#` 表示层级，一级到六级：

```markdown
# 一级标题
## 二级标题
### 三级标题
```

## 文本样式

- **加粗**：`**文本**`
- *斜体*：`*文本*`
- ~~删除线~~：`~~文本~~`
- `行内代码`：`` `代码` ``

## 代码块与语法高亮

使用三个反引号包裹，并指定语言即可启用语法高亮：

```python
def fibonacci(n: int) -> int:
    """计算斐波那契数列第 n 项"""
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

print(fibonacci(10))  # 55
```

## 表格

```markdown
| 列 A | 列 B |
|------|------|
| 内容 | 内容 |
```

## 引用

> 引用块适合突出显示提示、警告或重要结论。

## 链接与图片

```markdown
[链接文字](https://example.com)
![图片描述](/images/example.png)
```

## 任务列表

- [x] 完成写作
- [ ] 校对与发布

## 结语

Markdown 的魅力在于**用最简单的语法专注内容**。掌握以上语法，足以应对绝大多数技术写作场景。
