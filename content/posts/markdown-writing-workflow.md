---
title: "Markdown 写作工作流：从笔记到发布"
slug: "markdown-writing-workflow"
date: 2026-03-08
draft: false
description: "用纯文本搭建一套从灵感收集、笔记整理到一键发布博客的写作工作流，让内容创作不再被工具绑架。"
tags: ["Markdown", "写作", "Hugo", "自动化"]
categories: ["办公效率"]
---

## 为什么灵感总是留不住

很多人的写作痛点不是"不会写"，而是"写好的东西找不到"。灵感迸发的瞬间往往在通勤路上、会议间隙或睡前，等你打开一个厚重的文档工具、点开层层菜单、选好模板，那股劲儿早就散了。更糟的是，内容散落在备忘录、微信收藏、几个不同的笔记软件里，想把它们拼成一篇完整文章时，光是收集素材就要花掉半小时。

另一个隐形成本是格式。用富文本工具写作时，你会不自觉地陷入调字体、对齐、配色的陷阱，工具越好用，越容易把"写作"变成"排版"。而当你需要把同一份内容发到博客、发到内部文档、再摘录成周报时，又要重复格式化三遍。这些重复的、低价值的动作，正是效率杀手。

纯文本写作正是为解决这两个问题而生：它让你专注于内容本身，并且天然可迁移、可版本化、可自动化。

## 三层笔记结构：收件箱、项目、归档

我采用一个极简的三层目录结构，核心思想来自 GTD 的"收集—处理—组织"流程。所有笔记都是 `.md` 文件，用文件夹表达状态：

```bash
notes/
├── inbox/          # 收件箱：一切未经处理的灵感，随手记，不求工整
├── projects/       # 进行中：正在写的文章、正在推进的专题
│   └── markdown-writing-workflow/
│       ├── draft.md
│       └── assets/
├── reference/      # 资料库：摘录、读书笔记、别人的好例子
├── archive/        # 归档：已完成或暂时搁置的内容
└── templates/      # 模板与片段：复用块
```

规则很简单：**新想法一律先进 `inbox/`，不分类、不命名、不纠结**。每天或每两天集中处理一次收件箱：能立刻写的就移到 `projects/`，只是素材的移到 `reference/`，没价值的直接删掉。文章发布后，整个项目目录移入 `archive/`。这样一来，你的"待办"永远只需盯着 `projects/`，认知负担被降到最低。

这个结构的最大好处是**与工具无关**——文件夹和纯文本任何编辑器都能打开，哪怕十年后换了软件，内容依旧完整可读。

## 用模板和片段消灭重复劳动

每次新建文章，我都从同一份模板开始。它把 front matter、结构骨架、待办清单都预先写好，我只需要往里填内容：

```markdown
---
title: ""
slug: ""
date: {{date}}
draft: true
description: ""
tags: []
categories: ["办公效率"]
---

<!-- 写作 checklist
[ ] 场景引入（痛点 2 段）
[ ] 核心方法讲解
[ ] 可运行的代码示例
[ ] 最佳实践与踩坑
[ ] 小结
-->

## 场景引入

## 核心方法

## 实操示范

## 最佳实践与踩坑

## 小结
```

配合编辑器的片段（snippet）功能，常用的代码块、引用块、表格都能用几个字母展开。我在 VS Code 里把上面这份模板绑定为 `post` 片段，输入 `post` + Tab 即可生成。模板与片段的价值在于：**把"decide what to write"和"how to format it"两件事彻底分离**，写作时大脑只需处理前者。

## 从笔记到博客：一键发布脚本

真正的效率跃迁发生在发布环节。我的博客基于 Hugo，文章放在 `content/posts/`。下面这个脚本完成三件事：把草稿的 `draft` 改为 `false`、把文件从项目目录复制到博客目录、并按当前日期校准时间：

```python
#!/usr/bin/env python3
"""publish.py — 将 projects/<slug>/draft.md 发布到 Hugo content/posts/"""
import re
import shutil
from datetime import date
from pathlib import Path

ROOT = Path.home() / "notes"
POSTS = Path.home() / "blog/content/posts"

def publish(slug: str) -> None:
    src = ROOT / "projects" / slug / "draft.md"
    if not src.exists():
        raise SystemExit(f"草稿不存在: {src}")

    text = src.read_text(encoding="utf-8")
    # 把 draft: true 改成 false
    text = re.sub(r"^draft:\s*true", "draft: false", text, count=1, flags=re.M)
    # 没有 date 就补上今天
    if not re.search(r"^date:", text, flags=re.M):
        text = re.sub(
            r"^(slug:\s*.*)$",
            rf"\1\ndate: {date.today().isoformat()}",
            text, count=1, flags=re.M,
        )

    dst = POSTS / f"{slug}.md"
    dst.write_text(text, encoding="utf-8")
    # 同时归档源笔记
    shutil.move(str(src.parent), str(ROOT / "archive" / slug))
    print(f"已发布 -> {dst}")

if __name__ == "__main__":
    import sys
    publish(sys.argv[1])
```

运行 `python publish.py markdown-writing-workflow`，草稿即刻出现在博客目录，原项目自动归档。整个流程不到一秒，且完全可预测。这正是纯文本的优势——内容就是数据，数据就能被程序处理。你甚至可以在此基础上接入 `git commit && git push`，实现提交即上线。

## 双向链接与标签

当笔记积累到几百篇，文件夹的树状结构就不够用了，因为一篇笔记往往属于多个主题。我用两个手段补足：**标签**表达"这是什么"，**双向链接**表达"这篇和那篇有关"。在 Obsidian 等工具里，`[[另一篇笔记]]` 会生成反向链接，帮你发现意料之外的关联。标签则保持克制——我只用不超过十几个"领域级"标签（如 `#写作` `#效率`），避免陷入标签泛滥等于没有标签的困境。

## 最佳实践与踩坑

几点经过实践的经验。第一，**收件箱一定要定期清空**，否则它会变成第二个垃圾堆，建议设一个每周固定时间处理。第二，**slug 一旦发布就不要改**，它是文章的永久身份，改了会破坏外部链接，所以最好在草稿阶段就定好英文小写连字符格式。第三，**图片等资源放进项目自己的 `assets/` 目录**，用相对路径引用，迁移时不会丢图。第四，**不要在写作时反复调整模板**，模板是服务于产出的，够用就好，过度打磨模板本身就是一种拖延。第五，**定期用 git 对整个 `notes/` 目录做版本管理**，你会感谢那个能找回三天前被删段落的自己。

## 小结

这套工作流的核心不是某个神奇工具，而是三个原则：用纯文本让内容自由流动，用目录结构把决策前置，用脚本把重复动作自动化。写作最难的部分永远是从零到一的表达，把其余环节的摩擦降到最低，你才能把精力留给真正重要的思考。从今天起，先把你的下一个灵感丢进 `inbox/`，剩下的交给流程。
