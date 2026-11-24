---
title: "Kubernetes 排障手记：Pod 起不来的十个原因"
slug: "k8s-pod-troubleshooting"
date: 2026-05-18
draft: false
description: "从 Pod 状态机出发，系统梳理 Pending、ImagePullBackOff、CrashLoopBackOff、OOMKilled 等常见故障的诊断思路与 kubectl 命令。"
tags: ["Kubernetes", "Pod", "排障", "kubectl"]
categories: ["运维部署"]
---

## 又一个 Pod 卡在 Pending

"Pod 创建好了，但一直起不来"是 K8s 日常运维里最高频的求助。有人第一反应是删了重建，重试几次运气好能起来，但根因没消除，下次还会犯。K8s 的复杂在于一个 Pod 从提交到 Running 要经过调度、拉取镜像、创建容器、启动探针、就绪探针等多个环节，任何一步失败都会卡住，而表象往往只是一个冷冰冰的状态词。

排障的正确姿势是先看状态，再按状态对应的失败阶段去查。状态是 K8s 给你的最有价值的线索，它把问题范围直接缩小到某个环节。忽略状态盲目翻日志，等于放弃导航。

## 读懂 Pod 状态机

常见状态及其含义：

- **Pending**：还没调度到节点，或镜像还没拉下来。看 Events 找原因。
- **ImagePullBackOff / ErrImagePull**：镜像拉取失败——名字错、标签不存在、私有仓库没配 imagePullSecrets。
- **CrashLoopBackOff**：容器反复启动又退出，通常是应用启动即崩。
- **OOMKilled**：容器内存超限被内核杀掉，检查 limits。
- **CreateContainerConfigError**：引用了不存在的 ConfigMap/Secret。
- **ContainerCreating**：卡在创建中，多是存储卷挂载或网络插件问题。
- **Evicted**：节点资源不足被驱逐。
- **Init:CrashLoopBackOff / Init:Error**：初始化容器失败，主容器压根没启动。

## 诊断命令清单

一套自上而下的排查流程：

```bash
# 1. 看状态与事件，Events 里藏着根因
kubectl get pod -n <ns> -o wide
kubectl describe pod <pod> -n <ns>

# 2. 看容器日志，多容器要指定 -c，上一次崩溃用 -p
kubectl logs <pod> -n <ns> -c <container>
kubectl logs <pod> -n <ns> -c <container> --previous

# 3. 进入容器现场排查（没有 shell 时可 kubectl debug）
kubectl exec -it <pod> -n <ns> -- sh

# 4. 看集群事件流，按时间升序排列，最近的在最后
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -20

# 5. YAML 层面核对资源配置
kubectl get pod <pod> -n <ns> -o yaml
```

`describe` 输出里的 Events 段是金矿：调度失败会写 "0/5 nodes are available"，镜像失败会写 "Failed to pull image"，探针失败会写 "Liveness probe failed"。90% 的问题读到这里就有方向了。

## 典型配置片段：资源、探针与调度

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  imagePullSecrets:              # 私有仓库凭证，属于 PodSpec，与 containers 同级
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/web:v1.2.3
      resources:
        requests:                # 调度依据，太小会调度到资源不足的节点
          cpu: "200m"
          memory: "256Mi"
        limits:                  # 硬上限：memory 超了被 OOMKilled，cpu 超了只是被限流
          cpu: "1"
          memory: "512Mi"
      livenessProbe:             # 失败会重启容器
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:            # 失败只是摘流量，不重启
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
  nodeSelector:
    disktype: ssd                # 调度约束，标签不存在会 Pending
```

十个高频原因归结为几类：调度层（资源不足、nodeSelector/affinity 无匹配、污点容忍缺失）、镜像层（名称/标签错误、私有仓库认证）、配置层（ConfigMap/Secret 缺失、环境变量错）、运行层（应用崩溃、探针过于激进、启动慢于 initialDelay）、存储层（PVC 未绑定、挂载路径冲突）、资源层（limits 过低触发 OOM）。按这六类逐一对照状态，定位速度会快很多。

## 辅助诊断命令

诊断调度问题还要会看几个辅助命令。用 `kubectl describe node` 查看节点资源容量与污点（Taints），能解释"为什么 Pod 落不到这台机器"——常见的 `NoSchedule` 污点需要对应 toleration 才能调度。用 `kubectl top pod/node` 看实时资源使用，判断是 requests 设得过大造成浪费，还是 limits 过小导致频繁 OOM。Deployment 滚动更新卡住时，`kubectl rollout status` 会一直等待，此时 `kubectl describe rs` 往往能给出"达到 maxUnavailable 上限"这类线索。把调度、资源、滚动更新三层命令组合起来，多数 Pod 异常在进入容器之前就能锁定方向。最后别忘了事件（Event）默认只保留一小时，排障时若发现关键事件已过期，可接入事件导出器把 Event 持久化到日志系统，便于事后复盘。对于反复重启的 Pod，还要检查节点是否有磁盘压力或 PID 耗尽——这些节点级问题会表现为 Pod 层面的各种异常，却往往不在 Pod 配置里。

## 踩坑与最佳实践

liveness 探针别和 readiness 混用：把依赖外部服务的检查放进 liveness，网络抖动会触发无限重启。liveness 只查进程自身存活，readiness 才管流量摘挂。

`requests` 影响调度，`limits` 影响运行，两者都要设。OOMKilled 最常见的诱因是 memory limit 设得过低、贴着应用的实际峰值，甚至低于峰值；反过来，不设 memory limit 的容器没有 cgroup 上限，不会被 OOMKilled，风险转移到了节点层面——它会一路吃内存，直到被 kubelet 驱逐或拖垮整台机器。

Job 类容器崩溃可用 `restartPolicy: OnFailure`，长期跑的服务才用 `Always`，别让一次性任务陷入 CrashLoopBackOff。

排障结束记得清理调试时创建的临时容器或改动的副本数，避免遗留状态干扰后续判断。

## 小结

Pod 排障的本质是"按状态定位阶段，按阶段查证据"：Pending 看 Events 找调度与镜像，CrashLoopBackOff 看 `--previous` 日志，OOMKilled 查 limits。把 `get`/`describe`/`logs`/`events` 四条命令变成肌肉记忆，再理解 requests/limits、探针、调度约束这些配置如何影响生命周期，绝大多数"起不来"都能在几分钟内定位。K8s 不神秘，它只是把失败原因写在了状态和事件里——你要做的是学会阅读它们。
