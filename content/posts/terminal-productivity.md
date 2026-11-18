---
title: "终端效率工具链：tmux + fzf + ripgrep"
slug: "terminal-productivity"
date: 2026-04-19
draft: false
description: "把 tmux 的会话管理、fzf 的模糊搜索与 ripgrep 的极速查找组合成一套终端工作流，让键盘操作快到飞起。"
tags: ["终端", "tmux", "fzf", "ripgrep"]
categories: ["办公效率"]
---

## 你的时间都耗在了切换上

如果你的工作日常是：开五六个终端窗口，一个跑服务、一个看日志、一个改配置，然后在它们之间用鼠标来回点；想找一个函数定义，用 `grep -r` 等了十几秒；翻历史命令只能靠一直按方向键——那么问题不在于你敲字慢，而在于工具链陈旧。

终端效率的瓶颈通常有两个：一是**状态无法保持**，SSH 一断、终端一关，所有环境从头再来；二是**查找太慢**，无论是找文件、找内容还是找历史命令，都在打断你的心流。tmux、fzf、ripgrep 恰好各解决一类问题，而它们的真正威力在于**组合**——当你能把三件事串成一条命令、一次按键完成时，效率提升是质变而非量变。

这三款工具都是 Unix 哲学的典范：单一职责、可组合、配置即代码。下面我按"各司其职 → 配置 → 组合工作流"的顺序讲清楚。

## tmux：让会话永不丢失

tmux 是终端复用器。它在后台维护一个或多个会话（session），每个会话可包含多个窗口（window），窗口又能切成多个窗格（pane）。即使你关闭终端，会话仍在服务器/后台运行，下次连上 `attach` 即可回到原样。

最常用的几个操作：

```bash
tmux new -s work          # 新建名为 work 的会话
tmux ls                   # 列出所有会话
tmux attach -t work       # 重新接入（可简写 tmux a -t work）
```

在 tmux 内部，所有操作由前缀键（默认 `Ctrl-b`）加一个按键触发：

| 快捷键 | 作用 |
|-------|------|
| `Ctrl-b d` | 脱离当前会话（后台继续运行） |
| `Ctrl-b c` | 新建窗口 |
| `Ctrl-b n` / `p` | 切到下一个 / 上一个窗口 |
| `Ctrl-b %` | 左右分屏 |
| `Ctrl-b "` | 上下分屏 |
| `Ctrl-b 方向键` | 在窗格间移动 |
| `Ctrl-b z` | 当前窗格最大化/还原 |

一份顺手的 `~/.tmux.conf` 能显著改善体验，比如把前缀键改成更省力的 `Ctrl-a`，开启鼠标支持，并定义更直觉的分屏键：

```tmux
# ~/.tmux.conf
set -g prefix C-a
unbind C-b
bind C-a send-prefix

set -g mouse on                 # 鼠标可点击、拖动分屏
setw -g mode-keys vi            # 复制模式用 vi 键位
set -g base-index 1             # 窗口编号从 1 开始

# 用 | 和 - 分屏，比默认的 % 和 " 直观
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"

# 重启配置
bind r source-file ~/.tmux.conf \; display "配置已重载"
```

`-c "#{pane_current_path}"` 让新窗格继承当前目录，这个细节非常实用。

## ripgrep：毫秒级内容搜索

`rg` 是 `grep` 的现代替代品，默认递归、默认忽略 `.gitignore` 里的文件、默认显示行号、支持正则，而且快得离谱（Rust 编写）：

```bash
rg "TODO"                       # 在当前目录搜索
rg -i "error" logs/             # 忽略大小写
rg -n "def parse" -g '*.py'     # 只在 Python 文件里搜，显示行号
rg --files | wc -l              # 列出所有被追踪的文件
rg -l "config" src/             # 只输出包含匹配的文件名
```

几个高频参数：`-i` 忽略大小写，`-n` 显示行号，`-l` 只列文件名，`-g` 按 glob 过滤文件，`-C 3` 显示上下文各 3 行。把 `rg` 设成默认搜索工具，你会发现以前用 `grep -r` 翻半天的活儿现在一闪而过。

## fzf：把一切都变成模糊搜索

fzf 是一个通用的模糊查找器。给它一个列表，它提供交互式界面让你边打字边筛选。它本身不搜索，但能和任何命令组合。先做基础配置（以 zsh 为例）：

```bash
# ~/.zshrc
# 让 fzf 用 rg 生成候选文件，体验比 find 好得多
export FZF_DEFAULT_COMMAND='rg --files --hidden --glob "!.git"'
export FZF_DEFAULT_OPTS='--height 40% --layout=reverse --border'

# 激活 Ctrl-R 搜索历史命令、Ctrl-T 选文件、Alt-C 进目录
source <(fzf --zsh)

# 自定义：Ctrl-G 用 fzf 选文件并用 vim 打开
bindkey -s '^g' 'vim $(fzf)\n'
```

配好后，`Ctrl-R` 会弹出一个可实时过滤的历史命令列表，再也不用狂按方向键；`vim $(fzf)` 让你先可视化地挑文件再打开。

## 三者组合：一条命令的工作流

真正的高效来自组合。举两个典型场景。

**场景一**：在 tmux 的一个窗格里用 fzf 选文件、另一个窗格实时 `rg` 该文件的关键字。借助 fzf 的 `--preview` 可以一边选一边看内容：

```bash
# 选文件并在右侧预览前 30 行
fzf --preview 'head -30 {}'
```

**场景二**：批量搜索并进入文件。把 `rg -l` 的结果喂给 fzf，选中后用 vim 打开并跳到匹配行：

```bash
# 搜索所有含 "TODO" 的文件，fzf 选择后打开
rg -l "TODO" | fzf --preview 'rg -n "TODO" {}' | xargs -r vim
```

配合 tmux，你可以把编辑器、搜索、日志监控分在固定窗格，形成"一眼看全、手不离键盘"的工作台。很多人会写一个脚本，用 `tmux send-keys` 一键搭建整套布局，这就把环境准备也自动化了。一个典型做法是：建好三个窗格，分别启动代码编辑器、运行开发服务器、实时 `tail -f` 日志，下次上班运行一条命令，熟悉的工作台瞬间就位。这种"环境即代码"的思路，与基础设施即配置一脉相承——把重复的准备工作固化成脚本，既省时间又不会遗漏步骤。

## 最佳实践与踩坑

第一，`FZF_DEFAULT_COMMAND` 里用 `--glob "!.git"` 排除了版本库目录，否则候选列表会被 `.git` 里的文件淹没。第二，`rg` 默认尊重 `.gitignore`，如果要搜被忽略的文件需加 `--no-ignore`，搜隐藏文件加 `--hidden`。第三，tmux 的 `prefix` 改成 `Ctrl-a` 后，在某些原本用 `Ctrl-a` 的场景（如行首）会冲突，需要 `bind C-a send-prefix` 再按一次来传递。第四，在 tmux 里鼠标滚动需要 `set -g mouse on`，否则复制模式会打断。第五，fzf 依赖终端的真彩色支持，远程 SSH 时若显示异常，设置 `export TERM=xterm-256color` 通常能解决。

## 小结

tmux 管"状态"，让工作环境持续存在；ripgrep 管"找得快"，让内容搜索不再等待；fzf 管"选得准"，让交互选择行云流水。单独用，每个都能提速；组合起来，则形成一套以键盘为核心、几乎无需鼠标的终端工作流。建议你分三步引入：先装 ripgrep 替换掉 `grep`，再配好 fzf 的 `Ctrl-R`，最后花半小时调一份 tmux 配置。一周之后，你就回不去从前了。
