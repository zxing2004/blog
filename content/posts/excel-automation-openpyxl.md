---
title: "用脚本批量处理 Excel：openpyxl 实战"
slug: "excel-automation-openpyxl"
date: 2026-03-22
draft: false
description: "从重复的手工表格操作中解脱出来，用 Python 的 openpyxl 完成读写、清洗、合并与报表自动生成。"
tags: ["Python", "Excel", "openpyxl", "自动化"]
categories: ["办公效率"]
---

## 被表格支配的周三

每到月末，团队里总有一天被 Excel 彻底占据：十几个同事交上来的销售明细散在不同的 `.xlsx` 里，表头还各不相同，有人把日期写成 `2026/3/1`，有人写成 `3月1日`，还有人在合计行里混进了备注文字。你要把它们拼成一张总表、去重、按地区汇总、套上公司模板、再拆成各部门的分表——一套操作下来，半天就没了，而且眼睛一花就会出错。

更隐蔽的问题是**不可重复**。手工做完这个月，下个月还得从头再来一遍。而脚本的价值恰恰在这里：把一次性的痛苦，变成一次编写、长期复用的资产。下次 data 一到，运行一条命令，几秒钟出结果，还能顺手生成日志供核对。

Python 生态里处理 Excel 的库不少，但如果你的文件是 `.xlsx` 且需要保留样式、公式、合并单元格，`openpyxl` 是最稳妥的选择。它不依赖 Microsoft Excel，纯 Python 实现，跨平台可用。

有人会问，处理表格为什么不用 pandas？pandas 确实擅长数据分析与聚合，但它的读写过程会丢失单元格样式、公式、合并单元格、批注等 Excel 特性，导出的文件常常"面目全非"。当你的产物需要直接发给同事查看时，openpyxl 对格式的完整保留是 pandas 难以替代的。两者并不冲突，实践中常见的分工是：用 pandas 完成复杂的数据清洗与统计，用 openpyxl 做最终的格式美化与输出，各取所长。按这个分工写脚本，代码更短，出错也更少。

## 基础：读写工作簿

先建立心智模型：一个工作簿（Workbook）包含多个工作表（Worksheet），工作表由单元格（Cell）组成。`openpyxl` 的 API 几乎就是对这个模型的直接映射：

```python
from openpyxl import Workbook, load_workbook

# 创建一个新工作簿
wb = Workbook()
ws = wb.active
ws.title = "销售明细"

# 三种写值方式
ws["A1"] = "日期"
ws.cell(row=1, column=2, value="地区")
ws.append([2026, "华东", 12500])   # 追加一行

# 保存
wb.save("output/demo.xlsx")

# 读已有文件
wb2 = load_workbook("output/demo.xlsx", data_only=False)
for row in wb2["销售明细"].iter_rows(values_only=True):
    print(row)
```

注意 `data_only` 参数：设为 `False`（默认）时读取的是公式本身，设为 `True` 时读取公式上一次计算出的缓存值。这个区别在读取别人带公式的表时经常被坑——如果文件从未被 Excel 打开计算过，缓存值可能是 `None`。

## 样式与公式

报表要能直接给人看，样式少不了。`openpyxl` 用几类样式对象控制外观，且样式是可复用的：

```python
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

header_font = Font(name="微软雅黑", bold=True, color="FFFFFF")
header_fill = PatternFill("solid", fgColor="4472C4")
thin = Side(style="thin", color="D9D9D9")
border = Border(left=thin, right=thin, top=thin, bottom=thin)

for cell in ws[1]:                      # 第一行表头
    cell.font = header_font
    cell.fill = header_fill
    cell.alignment = Alignment(horizontal="center")
    cell.border = border

ws.freeze_panes = "A2"                  # 冻结首行
ws.column_dimensions["A"].width = 14

# 写入求和公式：C 列放公式，交给 Excel 计算
ws["C10"] = "=SUM(C2:C9)"
```

经验上，**样式对象要定义在循环外复用**，给成千上万个单元格逐个 new 一个 `Font` 会显著拖慢速度，而且生成的 XML 更臃肿。

## 完整实战：批量生成部门报表

下面是一个贴近真实需求的脚本：读取一个总表，按"地区"分组清洗数据，为每个地区生成一个独立工作表并写入汇总，最后输出一份带样式的报表文件。它是本章最值得收藏的部分——稍作修改就能套用到大多数"一表拆多表"场景。

```python
#!/usr/bin/env python3
"""按地区拆分总表并生成带样式的汇总报表。"""
from collections import defaultdict
from datetime import datetime
from pathlib import Path

from openpyxl import Workbook, load_workbook
from openpyxl.styles import Font, PatternFill, Alignment

SRC = Path("data/总表.xlsx")
DST = Path("output/地区报表.xlsx")

def normalize_date(v):
    """把多种日期写法统一为 date 对象。"""
    if isinstance(v, datetime):
        return v.date()
    for fmt in ("%Y/%m/%d", "%Y-%m-%d", "%m月%d日"):
        try:
            return datetime.strptime(str(v), fmt).date()
        except ValueError:
            continue
    return None

def main():
    src = load_workbook(SRC, data_only=True).active
    rows = list(src.iter_rows(min_row=2, values_only=True))

    groups = defaultdict(list)
    for date_v, region, amount in rows:
        if region is None or amount is None:
            continue                      # 清洗：丢弃无效行
        groups[str(region).strip()].append((normalize_date(date_v), region, float(amount)))

    out = Workbook()
    out.remove(out.active)
    head_font = Font(bold=True, color="FFFFFF")
    head_fill = PatternFill("solid", fgColor="2F5597")

    for region, items in sorted(groups.items()):
        ws = out.create_sheet(title=region[:31])   # sheet 名最长 31 字符
        ws.append(["日期", "地区", "金额"])
        for c in ws[1]:
            c.font, c.fill = head_font, head_fill
            c.alignment = Alignment(horizontal="center")
        for item in items:
            ws.append(item)
        total_row = ws.max_row + 1
        ws.cell(total_row, 2, "合计")
        ws.cell(total_row, 3, f"=SUM(C2:C{total_row - 1})").font = Font(bold=True)
        ws.freeze_panes = "A2"

    DST.parent.mkdir(parents=True, exist_ok=True)
    out.save(DST)
    print(f"生成 {len(out.sheetnames)} 个地区报表 -> {DST}")

if __name__ == "__main__":
    main()
```

脚本里几个细节值得注意：工作表名有 **31 字符上限**且不能包含 `\/:*?[]`，所以做了截断；`defaultdict` 让分组代码干净利落；清洗环节显式跳过空值，避免脏数据污染合计；日期归一化函数用循环尝试多种格式，容错性强。

## 大文件性能：read_only 与 write_only

当文件达到几十万行，默认的加载模式会把整个工作簿读进内存，可能吃掉几个 GB。这时要切换到流式模式：

```python
# 只读模式：迭代器方式读取，内存占用恒定
wb = load_workbook("big.xlsx", read_only=True, data_only=True)
for row in wb.active.iter_rows(values_only=True):
    process(row)
wb.close()

# 只写模式：只能 append，边写边落盘
wb = Workbook(write_only=True)
ws = wb.create_sheet("数据")
for i in range(1_000_000):
    ws.append([i, f"item-{i}"])
wb.save("big-out.xlsx")
```

代价是功能受限：只读模式下不能随机访问单元格、样式信息可能不全；只写模式下只能用 `append`，不能回头修改。这是典型的**用功能换内存**的取舍，大文件场景非常划算。

## 最佳实践与踩坑

第一，**永远不要覆盖原始文件**，输出到新路径，原始数据是最后的退路。第二，公式的缓存值问题：`data_only=True` 读到的值只在 Excel 保存过才存在，脚本生成的公式文件若立刻用 `data_only=True` 读会得到 `None`，需要权衡。第三，合并单元格后只有左上角单元格有值，读取其他位置会得到 `None`，处理前要先 `unmerge` 或只取锚点。第四，`openpyxl` **不支持 `.xls` 老格式**，遇到它需先用其他工具转换。第五，中文路径和文件名在现代版本里没问题，但极老的环境中建议先 `cd` 到目录用相对路径。

## 小结

用 `openpyxl` 处理表格，本质上是把"人肉点击"翻译成"数据变换"。一旦你习惯了用 `load_workbook` 和 `iter_rows` 思考，那些曾经令人生畏的月末表格就会退化成一段可测试、可复用、可定时运行的普通代码。从一个小任务开始——比如先写个脚本自动给周报套表头——你会很快体会到这种转变带来的自由。
