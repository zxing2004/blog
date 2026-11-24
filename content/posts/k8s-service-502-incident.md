---
title: "Service 间歇性 502：一个就绪探针配置引发的血案"
slug: "k8s-service-502-incident"
date: 2026-06-17
draft: false
description: "6 月大促前压测，网关侧偶发 502，直连 Pod 却永远是 200。追下去发现是一组照抄模板的就绪探针参数，加上缺失的优雅退出，一起在发布窗口里撕开了一个口子。"
tags: ["Kubernetes", "readinessProbe", "优雅退出", "故障复盘"]
categories: ["运维部署"]
---

## 故障背景

6 月中旬的大促压测周，订单履约服务第一次暴露了问题。那天下午压测组把 QPS 拉到平时的三倍，网关侧开始零星报 502，比例不高，稳定在 8%~12%，但持续不断，压测报告很难看。

这套服务跑在自建的 Kubernetes 1.28 集群上，Deployment 八个副本，前面是 ClusterIP 类型的 Service，再由 ingress-nginx 把流量导进来。服务是 Java 写的，启动要加载一大堆本地字典和缓存，从容器起来到真正能接流量大概八十秒。压测目标是确认大促当天扛得住，所以第一反应是怀疑容量：副本不够、连接池打满、还是 GC 在捣乱。

一个关键线索被我们忽略了很久：502 **只在发布窗口里出现**。稳定态压了半小时，一条 502 都没有；只要配合压测做一次滚动发布，错误就开始冒。

## 异常现象

我们按层做了对比。同一个 `/health` 接口，三个路径的结果完全不同：

```bash
# 1) 直连某个 Pod IP
$ for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " http://10.244.3.17:8080/health; done; echo
200 200 200 200 200 200 200 200 200 200

# 2) 走 ClusterIP Service
$ for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " http://order-svc.default.svc:8080/health; done; echo
200 200 200 200 200 200 200 200 200 200

# 3) 走 Ingress
$ for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code} " -H "Host: order.example.com" http://10.96.14.8/health; done; echo
200 200 502 200 200 200 200 200 502 200 200 200 200 200 200 200 200 200 200 200
```

这个结果本身就排除了一大半方向。直连 Pod 是 200，说明应用进程是健康的；走 ClusterIP 也是 200，因为 kube-proxy 只把流量转给 Ready 的 Pod，列表里本来就没有"不该接流量"的地址；只有经过 Ingress 才出错。

**这里有个很值得记住的点：ClusterIP 这条路给不出 502。** kube-proxy 做的是 L4 的 DNAT，要么连上，要么连不上（连接被拒或超时），它不会生成 HTTP 状态码。502 是 L7 代理才能产生的东西，所以它必然来自 ingress-nginx，来源于"nginx 连不上上游"。

去看 ingress-nginx 的错误日志，形态完全一致：

```text
2026/06/17 15:12:41 [error] 31#31: *88213 connect() failed (111: Connection refused) while connecting to upstream,
  client: 10.244.7.22, server: order.example.com, request: "GET /health HTTP/1.1",
  upstream: "http://10.244.5.9:8080/health", host: "order.example.com"
```

`Connection refused`，连的是 `10.244.5.9:8080`。也就是说 nginx 选了一个后端，拨过去，端口没人监听。

## 排查思路

现在问题很具体了：为什么 nginx 会往一个没有监听端口的 Pod 上打流量？

第一反应是 Endpoints 抖动。用 `-w` 盯了一会儿：

```bash
$ kubectl get endpoints order-svc -w
NAME        ENDPOINTS                                                     AGE
order-svc   10.244.3.17:8080,10.244.4.21:8080,10.244.5.9:8080 +6 more    42d
order-svc   10.244.3.17:8080,10.244.4.21:8080 +7 more                      42d
order-svc   10.244.3.17:8080,10.244.4.21:8080,10.244.5.9:8080 +6 more    42d
order-svc   10.244.3.17:8080,10.244.4.21:8080,10.244.6.33:8080 +6 more   42d
```

`10.244.5.9` 被摘掉又加回来，几秒一个来回。这就是"反复摘挂"。

拉一个 Pod 的描述看探针：

```bash
$ kubectl describe pod order-7d9f4c8b9-x2kqn | sed -n '/Readiness:/,/QoS Class:/p'
    Readiness:  http-get http://:8080/health delay=2s timeout=1s period=2s #success=1 #failure=3
...
QoS Class:       Burstable
```

`delay=2s`、`timeout=1s`、`period=2s`、`failure=3`。两个问题叠在这四个数字里：

一是**没有 startupProbe**。应用要八十秒才真正就绪，探针两秒后就开始查，冷启动期间必然全是失败——这部分勉强算"行为正确"，但它让每次发布的可用副本数长时间少一个。

二是 **`timeoutSeconds: 1` 太短，而且只在压测时暴露**。`/health` 和业务请求共用同一个 Tomcat 线程池，三倍 QPS 下健康检查偶尔要排到一秒以上。一次超时算一次失败，三次连续失败（不到六秒）就把一个**完全健康**的 Pod 摘出 Endpoints；下一轮又成功，再挂回去。这才是抖动的主因——它和"启动慢"没关系，是探测本身在负载下不稳。

查到这一步，502 的链路已经能串起来了，但还差最后一环：Endpoints 里摘掉一个 Pod，本来只会少一份可用容量，为什么会导致 `Connection refused`？

答案在优雅退出。看 Deployment 的完整定义：

```bash
$ kubectl get deploy order -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}'
$ kubectl get deploy order -o jsonpath='{.spec.template.spec.containers[0].lifecycle}'
```

两个都是空的——`terminationGracePeriodSeconds` 用默认 30 秒，**没有任何 `preStop` 钩子**。pod 被删除时，kubelet 立刻给容器发 SIGTERM，Spring Boot 收到后进入关闭流程，第一步就是停止接受新连接、关闭监听端口。而这时 Endpoints 的变更还没同步到所有 ingress-nginx 实例——它自己维护一份 upstream 列表，靠 watch + 几秒的收敛延迟。这几秒里，nginx 手里仍然是旧列表，拨过去，端口已经关了，`Connection refused`，502。

`maxSurge: 1, maxUnavailable: 0` 又把这个窗口拉长了：新 Pod 必须 Ready 之后才会回收旧 Pod，而新 Pod 因为探针问题要卡八十多秒，一次发布要拖很久，中间不断有 Pod 处在"正在关闭但还没从所有 nginx 的列表里消失"的状态。发布拖得越久，502 出现的机会越多。

## 处理过程

改的是三处，都不动业务代码。

第一，补 startupProbe，把"启动期"和"运行期"分开，避免冷启动阶段的失败被计入可用性判断：

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 30        # 5 × 30 = 150s，覆盖 80s 冷启动并留余量
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3         # 连续 15s 失败才摘流量，扛住瞬时抖动
  successThreshold: 1
```

注意 `timeoutSeconds` 从 1 提到 3：健康检查的阈值必须高于该接口在峰值负载下的 P99，否则它测的不是"服务是否可用"，而是"这一秒线程池忙不忙"。这一条比任何参数都重要。

第二，加优雅退出。让容器在收到 SIGTERM 后先"什么都不做"几秒，等 Endpoints 变更扩散出去、nginx 的 upstream 列表收敛完，再真正关端口：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 8"]
terminationGracePeriodSeconds: 60
```

第三，把健康检查接口从业务线程池里摘出去。Spring Boot 里给 actuator 单独配一个线程池或改用独立的 management port（`management.server.port: 8081`），让它不再和业务请求抢资源。这是治本的一步——前两条只是让探测更宽容，这条才让它不再受业务负载影响。

改完重新滚动发布，`kubectl get endpoints -w` 不再抖动，压测期间 502 归零。

## 根因分析

根因不是"探针参数写错了"这么简单，而是三个独立的问题恰好凑成了一个故障。

第一层，**探针阈值低于接口的真实延迟分布**。`timeoutSeconds: 1` 对一个平时 P99 只有几十毫秒的接口看起来绰绰有余，但压测时这个接口的 P99 会涨到一秒以上。用固定阈值去衡量一个动态延迟的接口，本身就不成立。探针应该测"进程是否还能响应"，而不是"响应得够不够快"。

第二层，**没有 startupProbe**。1.28 已经支持这个字段很久了，但模板里没有。没有它，冷启动阶段的失败和运行期的失败混在一起，探针的语义就模糊了。

第三层，也是最致命的：**没有优雅退出**。Kubernetes 的删除流程是并行的——发 SIGTERM、从 Endpoints 摘除、通知各节点更新转发规则，三件事同时发生，谁先到不确定。应用如果收到 SIGTERM 就立刻关监听，必然存在一段"已经不能服务、但还有代理在往这儿发流量"的窗口。`preStop` 的唯一作用就是人为把这段窗口错开。

顺带说一个我们当时的误解：一直以为 502 是"Service 转发到了不健康的 Pod"。实际上 kube-proxy 只会转发给 Ready 的 Pod，Service 这一层不会给你 502。真正的竞争发生在 L7 代理的本地配置和集群状态之间——这是两个不同步的数据源。

## 复盘总结

这次故障没有改一行业务代码，代价却实实在在。我们固化了四条。

第一，探针的 `timeoutSeconds` 必须大于接口在峰值负载下的 P99，而不是照着平时设。这条写进了发布 checklist。

第二，凡是启动超过 10 秒的服务，一律配 startupProbe，`failureThreshold × periodSeconds` 按最坏启动时间倒推。

第三，**所有服务必须有 preStop 和显式的 terminationGracePeriodSeconds**，`preStop` 睡 5~10 秒是标准动作。这条我们做成了准入校验，Chart 模板里强制注入。

第四，压测必须覆盖"发布中"的状态。这次的 502 只在滚动更新窗口里出现，常规的稳态压测根本测不到——这也解释了为什么它躲过了之前所有的压测。

故障复盘会上有人说，这算是被模板坑了一次。我倒觉得模板没错，错的是我们把它当默认值用了两年，从没问过一句：这个数字，到底是为谁准备的？把"能用"换成"适用"，大概就是这次复盘最该留下来的东西。
