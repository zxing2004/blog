---
title: "Markdown 写作速查与进阶技巧"
slug: "markdown-cheatsheet"
date: 2026-04-05
draft: false
description: "一份面向技术写作的 Markdown 速查表，涵盖标题、代码块、表格、脚注等常用语法。"
tags: ["Markdown", "写作", "Hugo"]
categories: ["工具与效率"]
aliases:
  - "/posts/markdown-写作速查与进阶技巧/"
---

## 标题

使用 `#` 表示层级，最多到六级：

```markdown
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

实践上一篇文章顶多用到三级，再往下读者就该迷路了——层级深了，说明该拆小标题或拆文章。

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

## 脚注与引用式链接

技术写作里有两个"进阶但极省事"的语法。脚注适合放数据来源、版本说明：

```markdown
这里的吞吐数据来自官方基准[^1]。

[^1]: 数据出自 vLLM v0.6 的 benchmark 脚本，A100 80G × 8，batch=32。
```

引用式链接则把长 URL 挪到文末，正文里只留一个短标识，改链接时只改一处：

```markdown
这是[官方文档][docs]里的说法。

[docs]: https://example.com/docs "官方文档"
```

脚注能否渲染取决于主题与渲染器（GitHub、Typora、Hugo 的 Goldmark 默认都支持；部分静态站点生成器需要显式开启 `footnotes` 扩展），发布前最好本地看一眼效果。

## 任务列表

- [x] 完成写作
- [ ] 校对与发布

## 结语

Markdown 的魅力在于**用最简单的语法专注内容**。上面这些语法足以应对绝大多数技术写作场景；真遇到表达不了的结构（复杂表格、图注），再考虑嵌入 HTML 也不迟。
