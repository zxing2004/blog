---
title: "自动化你的重复工作：从 cron 到工作流"
slug: "automate-repetitive-tasks"
date: 2026-06-06
draft: false
description: "识别值得自动化的重复任务，用 cron 编排脚本、处理错误、发送通知，并建立幂等与日志机制，让机器替你值守。"
tags: ["自动化", "cron", "Shell", "运维"]
categories: ["办公效率"]
---

## 那些"每天都要做一遍"的事

每天上班第一件事，导出昨天的报表；每天下班前，把几个目录打包备份；每周一，汇总上周数据发到群里；每月初，清理过期日志。这些任务有一个共同特征：**规则明确、重复发生、没人愿意做**。它们不像创造性工作那样需要你，但又像影子一样每天准时出现，悄悄吞掉你的时间与注意力。

判断是否值得自动化，有个简单标准：一件事如果**做过三次以上、每次步骤基本一致、做错了能重来**，就该交给脚本。手工做这类事不仅慢，还会因为疲劳和分心而出错——而自动化脚本每次执行都一模一样，可靠且可预测。

cron 是 Unix 世界最经典的定时任务工具，配合 Shell 脚本，足以覆盖绝大多数个人和小团队的自动化需求。但真正能用得久的自动化，不止是"定时跑一条命令"，还要考虑错误处理、通知、幂等和日志。下面逐一讲清楚。

## cron 语法与 crontab

cron 用五字段表达式描述时间，依次是：分、时、日、月、周：

```bash
# ┌───────── 分钟 (0-59)
# │ ┌─────── 小时 (0-23)
# │ │ ┌───── 日期 (1-31)
# │ │ │ ┌─── 月份 (1-12)
# │ │ │ │ ┌─ 星期 (0-7, 0和7都是周日)
# │ │ │ │ │
# * * * * *  command-to-run

0  9  *  *  1-5   /opt/scripts/report.sh     # 工作日每天 9 点
*/5 *  *  *  *     /opt/scripts/check.sh     # 每 5 分钟
0  2  *  *  *      /opt/scripts/backup.sh    # 每天凌晨 2 点
30 18 *  *  5      /opt/scripts/weekly.sh    # 每周五 18:30
```

用 `crontab -e` 编辑当前用户的定时任务，`crontab -l` 查看。一份考虑了环境变量与日志的 crontab 长这样：

```bash
# /etc/crontab 或 crontab -e 内容
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
# 变量赋值行不能带行尾注释：cron 会把 "# 任务输出…" 整段当成邮箱地址
MAILTO=you@example.com

0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
0 9 * * 1-5 /opt/scripts/report.sh >> /var/log/report.log 2>&1
```

这里有个很隐蔽的坑：`MAILTO=you@example.com  # 注释` 看着像正常的配置加注释，但 crontab 只把"整行以 `#` 开头"当作注释，环境变量行是**取等号后全部内容**作为值。结果 MAILTO 变成了 `you@example.com  # 注释`，邮件自然发不出去，而且 cron 不会报错——它只是静默地失败。

注意 cron 默认环境非常干净，`PATH` 往往只有系统路径。脚本里用到 `python3`、`conda` 等自定义路径时，要么在 crontab 里显式设置 `PATH`，要么在脚本中用绝对路径——这是新手最常踩的坑。

## 一个可靠的定时备份脚本

先看一个完整例子：每天打包指定目录，按日期命名，保留最近 7 天，失败时发通知：

```bash
#!/usr/bin/env bash
# /opt/scripts/backup.sh —— 每日备份并轮转
set -euo pipefail

SRC="/data/app"
DST="/backup"
LOG="/var/log/backup.log"
KEEP=7

log() { echo "[$(date '+%F %T')] $*" | tee -a "$LOG"; }

log "开始备份 $SRC"
tar -czf "$DST/app-$(date +%F).tar.gz" -C "$SRC" . || {
  log "ERROR: 打包失败"
  exit 1
}

# 只保留最近 7 天：-mtime +6 表示"修改时间在 6×24 小时以前"，即第 8 天及更早
find "$DST" -name 'app-*.tar.gz' -mtime +"$((KEEP - 1))" -delete
log "备份完成，已清理超过 ${KEEP} 天的旧文件"
```

`-mtime` 的边界值得单独记一笔：它是按 24 小时为单位向下取整的，`-mtime +7` 删的是**满 7 天以上**、也就是第 8 天的文件，想"保留最近 7 天"得写 `+6`。差一天看着无害，直到某次恢复时发现唯一需要的那份刚好被删了。

`set -euo pipefail` 是 Shell 脚本的"安全带"：`-e` 遇错即停，`-u` 变量未定义即报错，`-o pipefail` 让管道中任何一环失败都算失败。把它写在每个脚本开头，能避免大量"静默失败"的诡异问题。`log` 函数用 `tee` 同时输出到终端和日志文件，排障时一目了然。

## 通知机制：让脚本主动告诉你结果

自动化最怕"静默失败"——任务挂了几天你都不知道。加一个 webhook 通知，成功失败都发条消息到群里。以企业微信机器人为例：

```bash
#!/usr/bin/env bash
# notify.sh —— 发送消息到 webhook
send() {
  local msg="$1"
  curl -sS -X POST 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY' \
    -H 'Content-Type: application/json' \
    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"$msg\"}}"
}

# 在报表脚本末尾调用
if /opt/scripts/report.sh; then
  send "报表生成成功：$(date +%F)"
else
  send "报表生成失败，请检查日志"
fi
```

邮件通知更简单：`echo "内容" | mail -s "标题" you@example.com`，配合 crontab 的 `MAILTO` 也能实现基础通知。选择哪种取决于团队习惯，关键是**异常一定要有回响**。通知的粒度也需要设计：只推送异常和关键结果，别让每条成功日志都刷屏——一旦消息变得频繁，人就会习惯性忽略，反而错过了真正的告警。

## 幂等与日志：自动化的两条护栏

**幂等**指同一任务执行一次和执行多次的结果相同。这很重要，因为脚本可能因网络抖动、手动补跑而重复触发。实现幂等的常见手段有两类：一是"处理前先检查状态"，比如备份前先判断当天文件是否已存在；二是用锁文件防止并发，工具 `flock` 很好用：

```bash
#!/usr/bin/env bash
# 用 flock 保证同一时刻只有一个实例运行
exec 9>/var/lock/report.lock
if ! flock -n 9; then
  echo "已有实例在运行，退出"; exit 0
fi

# 以下为任务主体
/opt/scripts/do_report.sh
```

`exec 9>lockfile` 打开文件描述符 9，`flock -n 9` 非阻塞加锁，拿不到锁就退出，从而避免两个实例互相踩踏。

**日志**要记录三要素：时间、做了什么、结果。像上面的 `log` 函数那样带时间戳输出，并区分 `INFO` 与 `ERROR` 级别。定期归档或轮转日志（可用 `logrotate`），别让它把磁盘撑爆。

## 最佳实践与踩坑

第一，**先在终端手动跑通脚本，再挂到 cron**，别指望调试定时执行的任务。第二，**脚本里一律用绝对路径**，因为 cron 的环境变量和你登录时完全不同。第三，**给脚本加上 `set -euo pipefail`**，让错误尽早暴露而不是悄悄继续。第四，**注意时区**，cron 使用系统时区，云服务器常默认是 UTC，和北京时间差 8 小时，部署前务必确认 `date` 输出。第五，**通知要克制**，只报异常和关键结果，别让成功消息刷屏，否则你会习惯性忽略它。第六，**敏感信息别写死在脚本里**，webhook key、数据库密码等用环境变量或配置文件管理，并限制文件权限。第七，**定期巡检**，每月看一眼日志，确认任务真的在按预期运行。

## 小结

自动化不是"设置完就忘"，而是把重复劳动交出去、把自己的注意力收回来。一条可靠的自动化链路是：识别高价值重复任务 → 写成带安全带的脚本 → 用 cron 定时触发 → 配置错误通知 → 用幂等和日志兜底。从最让你烦躁的那件事开始——哪怕只是每天自动导一份报表——把它做成第一个脚本。当机器开始替你值守，你会真切感受到时间被还给了自己。
