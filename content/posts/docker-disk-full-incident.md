---
title: "Docker 宿主机磁盘写满：一次日志风暴引发的雪崩"
slug: "docker-disk-full-incident"
date: 2026-02-19
draft: false
description: "一个容器的错误日志把 100G 数据盘写到 0 字节可用，拖垮了宿主机上所有容器和 CI 构建。复盘这场由日志风暴引发的连锁故障。"
tags: ["Docker", "日志轮转", "overlay2", "磁盘告警"]
categories: ["运维部署"]
---

## 故障背景

2 月中旬那次发布之后，我们一台专门跑应用容器的 Docker 宿主机出了问题。这台机器挂着一块 100G 的 SSD 作为 `/var/lib/docker` 数据盘，上面跑了三十多个容器，既有线上服务，也承担部分 CI 构建任务。平时磁盘使用率稳定在五成左右，没人觉得它会是风险点。

发布的是订单导出服务的一个小版本，改动很小——调整了一个下游接口的超时重试逻辑。上午九点二十上线，健康检查全绿，日志量也在正常范围。真正的变化是：新版本在下游持续超时时，每失败一次就把整个响应体原样打进错误日志，一条两 K 上下，然后立刻重试。上午业务量低，下游压力小，重试很少触发；到了下午高峰，比例上来了，日志曲线开始以肉眼可见的斜率往上爬，平均每秒将近一千条，也就是 1.5MB/s 左右。这个速度不刺眼，但它是单向的——十一个小时就是六十多 G。

晚上八点四十，监控开始报磁盘使用率飙升。等值班同学点开面板，曲线已经是一条接近垂直的直线。

## 异常现象

最先炸的是日志采集。Filebeat 报 `write error: no space left on device`，紧接着多个容器开始重启，健康检查连续失败。更麻烦的是连锁反应：镜像拉取全部失败，正在跑的 CI 任务纷纷报 `failed to register layer: no space left on device`，连开发同学本地的构建也没法把产物推上来了。

登上宿主机第一眼就是 `df -h`，数据盘已经彻底写满：

```bash
$ df -h /var/lib/docker
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       100G  100G     0 100% /var/lib/docker
```

100G 用了 100G，可用 0 字节。这个状态下任何需要落盘的操作都会失败，容器写文件、镜像解层、构建缓存，无一幸免。换句话说，这不是一个容器的问题，而是整台宿主机失去了写入能力。

## 排查思路

磁盘满了，先搞清楚是谁占的。`du` 拉一遍顶层目录，`containers` 占了七十八个 G，明显不正常：

```bash
$ du -sh /var/lib/docker/* | sort -h
521M    /var/lib/docker/buildkit
3.4G    /var/lib/docker/overlay2
 18G    /var/lib/docker/image
 78G    /var/lib/docker/containers
```

`containers` 目录下每个子目录对应一个容器，主要放的是 json-file 日志。继续往下钻，直接按目录大小排序，不用猜是哪个容器：

```bash
$ du -sh /var/lib/docker/containers/* | sort -h | tail -3
1.2G    /var/lib/docker/containers/9c1d.../9c1d...-json.log
2.8G    /var/lib/docker/containers/5f72.../5f72...-json.log
 62G    /var/lib/docker/containers/3f9a.../3f9a...-json.log
```

找到最大的那个，再用 inspect 反查它属于哪个容器（目录名就是容器 ID 前缀）：

```bash
$ docker inspect --format '{{.Name}} {{.Config.Image}} {{.LogPath}}' 3f9a1c8e2b40
/order-export-prod order-export:v2 /var/lib/docker/containers/3f9a.../3f9a...-json.log
$ tail -n 2 /var/lib/docker/containers/3f9a.../3f9a...-json.log | cut -c1-200
{"log":"ERROR retry failed: context deadline exceeded, resp={\"code\":500,\"msg\":\"upstream timeout\",\"trace\":\"...\"}\n","stream":"stderr","time":"2026-02-19T20:41:33.912Z"}
{"log":"ERROR retry failed: context deadline exceeded, resp={\"code\":500,\"msg\":\"upstream timeout\",\"trace\":\"...\"}\n","stream":"stderr","time":"2026-02-19T20:41:33.914Z"}
```

答案指向 `order-export-prod`，单个容器的日志文件 62G，比剩下所有东西加起来还多，而且时间戳显示它每秒还在新增上千行。新版本的重试逻辑在下游超时后没有退避、没有熔断，日志里又塞了完整响应体，每次失败都往盘里灌两 K。

## 处理过程

第一目标是先把空间腾出来，恢复写入。这里有个关键细节：不能直接 `rm` 日志文件。容器的 stdout 还通过文件描述符连着这个文件，删掉目录项后 inode 不会释放，`df` 依旧显示 100%。正确做法是把文件截断：

```bash
$ truncate -s 0 /var/lib/docker/containers/3f9a.../3f9a...-json.log
$ df -h /var/lib/docker
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       100G   38G   62G  38% /var/lib/docker
```

空间瞬间回到 62G，容器和 CI 陆续恢复。

但这只是止血——应用还在按每秒上千条的速度写，不回滚的话几个小时后又满。第二件事是回滚：把订单导出服务退到上一个版本，重试逻辑回到带退避的老实现，日志速率立刻降到正常水平。

第三件事才是配上限。改 `/etc/docker/daemon.json`，`max-size` 控制单个日志文件上限，`max-file` 控制保留份数，这样每个容器最多占 300M：

```bash
$ cat > /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
$ systemctl restart docker
$ docker info --format '{{.LoggingDriver}}'
json-file
```

这里有个必须说清的坑：`log-opts` 是**创建容器时**写进容器配置的，改 daemon.json 只对之后新建的容器生效，已经在跑的三十多个容器还是老配置。所以还得清理无用的镜像和构建缓存补回空间，并安排一次滚动重建，把所有存量容器换掉：

```bash
$ docker system prune -a --filter "until=168h" -f
Deleted Images:  14
Deleted Containers: 6
Total reclaimed space: 21.7GB
```

当晚协调存储把数据盘从 100G 扩到 200G，算是给重建留出缓冲。

## 根因分析

这次雪崩是三个缺口叠在一起的结果，缺任何一个都不会走到"整台宿主机失去写入能力"。

第一是应用侧：新版本的重试逻辑没有退避和熔断，下游一超时就陷入紧密循环；更要命的是把完整响应体打进了日志，单条体积从几十字节涨到两 K。同样的重试频率，日志体积被放大了三十倍。这是导火索。

第二是容器日志无上限。Docker 默认的 json-file 驱动不限制大小和数量，一个容器想写多少就写多少。我们从来没配过 `log-opts`，等于给日志风暴留了一条畅通无阻的通道。

第三是磁盘告警缺失。采集是有的，阈值没设，全靠人看面板。水位从 50% 爬到 100% 用了十一个小时，这期间没有任何人被通知。如果 80% 时有人被叫醒，那会儿业务还完全正常，代价会小得多。

还有一层值得单独点出来：**增长太"温和"反而更危险**。1.5MB/s 这种速度在监控曲线上看起来像一条平缓的斜线，不像 CPU 打满那样有戏剧性，很容易被当成"正常波动"。它是慢性的，但方向单一，最终一定会撞墙。

## 复盘总结

事后我们做了几件事。

全局层面，所有 Docker 宿主机统一下发 `daemon.json` 强制 `max-size` 和 `max-file`，新节点上线作为基线检查项；同时把"存量容器是否带 log-opts"做成一个巡检脚本，因为改配置不会影响已有容器，这一点最容易漏。

监控层面，给 `/var/lib/docker` 加了分级告警：70% 提醒、85% 严重、95% 电话；另外对 `containers` 目录单独设了一条**增长率**告警——绝对值慢不等于没问题，斜率才说明问题。

应用层面，要求所有重试逻辑必须带退避、熔断和最大次数；错误日志只打必要的上下文字段，禁止把完整响应体裁进日志，这条写进了 code review checklist。

最后一条写进了发布规范：涉及重试、超时、循环的改动，发布后必须观察半小时的**日志速率**，而不是只看健康检查。健康检查只告诉你服务活着，不告诉你它正在把磁盘写穿。我们还把"磁盘写满"做成 runbook 的第一个场景，把 truncate 日志、按目录定位大容器、清缓存这几步固化下来，下次不用在现场翻命令。
