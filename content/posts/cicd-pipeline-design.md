---
title: "CI/CD 流水线设计：从构建到灰度发布"
slug: "cicd-pipeline-design"
date: 2026-05-04
draft: false
description: "拆解 CI/CD 流水线的阶段划分、制品缓存、滚动/蓝绿/金丝雀部署与失败回滚，并给出可直接用的 GitLab CI 示例。"
tags: ["CI/CD", "GitLab CI", "灰度发布", "DevOps"]
categories: ["运维部署"]
---

## 流水线不是把脚本串起来那么简单

很多团队的"流水线"是一堆 shell 脚本的顺序堆砌：拉代码、编译、打包、scp 到服务器、重启。能跑，但每次发布都像开盲盒——构建结果不可复现、失败后不知道停在哪一步、出问题时无法快速回滚，更别提灰度。当发布频率从一周一次提到一天十几次，这种脆弱的链路会立刻崩溃。

一条成熟的流水线要解决三个问题：每次产出可追溯且一致的制品；质量门禁让有问题的代码进不了生产；发布过程可控可回滚。围绕这三点,把流水线切成明确的阶段，每个阶段职责单一、结果可缓存、失败可定位。

## 阶段划分与制品缓存

典型流水线分为四段：构建（compile/build）→ 测试（unit/integration）→ 镜像（package）→ 部署（deploy）。核心原则是"一次构建，多处部署"：同一个制品从测试环境逐级晋升到生产，杜绝"测试用的包和生产重新编译的包不是同一个"这种致命问题。

缓存是提速关键。依赖下载（npm/maven/go mod）往往占构建大半时间，应按依赖清单哈希做 key 缓存；Docker 层缓存也要利用好：

```yaml
# key 由 go.sum 的内容哈希决定：依赖一变 key 就变，旧缓存自然失效
cache:
  key:
    files:
      - go.sum
  paths:
    - .cache/go-build
    - go/pkg/mod
```

用 `key: files:` 而不是自己拼字符串，是因为 GitLab 会对列出的文件算内容哈希生成 key——写成 `key: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"` 这类形式，依赖升级了 key 却不变，缓存会一直命中旧依赖，正是"本地能编译、CI 上莫名报错"的常见来源。

制品（jar、二进制、镜像）应存入制品库（如 Harbor、GitLab Registry）并以 commit SHA 打标签，保证任何一次部署都能精确追溯到代码版本。

## 部署策略：滚动、蓝绿与金丝雀

- **滚动更新（rolling）**：逐个替换实例，资源省但新旧版本并存，回滚慢。
- **蓝绿部署（blue-green）**：两套环境同时存在，切换流量是一瞬间的原子操作，回滚快但资源翻倍。
- **金丝雀（canary）**：先放一小部分流量到新版本，观察指标正常再逐步放大，风险最小但对流量调度与可观测性要求高。

## 一份完整的 GitLab CI 示例

```yaml
stages: [build, test, package, deploy, rollback]

variables:
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: golang:1.22
  script:
    - go build -ldflags="-s -w" -o app .
  artifacts:
    paths: [app]
    expire_in: 1 week

unit-test:
  stage: test
  image: golang:1.22
  script:
    - go test -race -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out | tail -n 1    # 输出 total: (statements) 78.3%
  coverage: '/total:\s+\(statements\)\s+(\d+\.\d+)%/'

package-image:
  stage: package
  image: docker:24
  services: [docker:24-dind]
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin
    - docker build -t $IMAGE .
    - docker push $IMAGE

deploy-staging:
  stage: deploy
  script:
    - kubectl set image deploy/app app=$IMAGE -n staging
    - kubectl rollout status deploy/app -n staging --timeout=180s
  environment:
    name: staging

# 前提：app 与 app-canary 两个 Deployment 的 Pod 模板都带 app: web 标签，
# Service 的 selector 只写 app: web，于是两边 Pod 会被同一个 Service 选中，
# 流量按副本数近似分配。要精确百分比得上 Istio VirtualService 权重。
deploy-production:
  stage: deploy
  script:
    # 金丝雀：9 个稳定副本 + 1 个金丝雀副本 ≈ 10% 流量
    - kubectl scale deploy/app --replicas=9 -n production
    - kubectl set image deploy/app-canary app=$IMAGE -n production
    - kubectl scale deploy/app-canary --replicas=1 -n production
    - kubectl rollout status deploy/app-canary -n production --timeout=180s
    - ./scripts/verify-canary.sh   # 对比 canary 与 stable 的错误率、P99，异常则 exit 1
    # 校验通过，全量滚动并摘除金丝雀
    - kubectl set image deploy/app app=$IMAGE -n production
    - kubectl scale deploy/app --replicas=10 -n production
    - kubectl rollout status deploy/app -n production --timeout=300s
    - kubectl scale deploy/app-canary --replicas=0 -n production
  when: manual                     # 生产发布需人工确认
  environment:
    name: production
```

这里有个常见误解值得点明：`kubectl set image` **只换镜像，不切流量**。如果 app 和 app-canary 是两个互不相干的 Deployment、Service 只选其中一边，那"先发金丝雀"就只是多起了一份没人访问的 Pod，接着全量发布等于直接上生产。真正实现 10% 分流有两种做法：一是上面这种"两个 Deployment 共用 selector、按副本数近似"，够用但粒度粗；二是引入 Ingress 或 Istio 的权重路由，把 10%、30%、50% 写进配置，粒度精确但多一层依赖。

`kubectl rollout status` 是关键：它让流水线真正确认新版本就绪，而不是"命令提交成功"就算完成。

## 环境晋升与质量门禁

环境晋升要遵循"制品不变、配置变"的原则：同一份镜像通过修改配置（ConfigMap、环境变量）适配不同环境，而不是为每个环境重新构建。这样测试环境验证过的产物与生产完全一致，回滚时也只需切回旧配置与旧镜像标签，无需重新构建。质量门禁则应分层：单元测试最快、反馈最及时，放在最前；集成与契约测试居中；端到端 UI 测试最慢且脆弱，可作为非阻塞检查或定时运行。把最快的检查放在最前，让开发者几秒内拿到反馈，是提升研发体验的关键。流水线的每次运行还应保留日志与制品索引，满足审计与溯源要求。另外要警惕"流水线本地绿、线上红"的现象，根因往往是本地与 CI 环境的工具版本、系统依赖不一致。用容器作为构建环境、在 CI job 里指定统一镜像，能消除这类差异，让构建结果真正可复现。

## 失败回滚与最佳实践

回滚要自动化，不能依赖人肉记忆。一旦部署阶段超时或健康检查失败，应触发回滚任务：

```yaml
rollback:
  stage: rollback          # 必须独立成后置 stage，不能和 deploy 同 stage
  when: on_failure
  script:
    - kubectl rollout history deploy/app -n production | tail -n 3
    - kubectl rollout undo deploy/app -n production
    - kubectl rollout status deploy/app -n production --timeout=180s
```

`when: on_failure` 的语义是"**前面某个阶段**失败时触发"，它不看同一个 stage 里的兄弟 job。所以如果把 rollback 和 deploy-production 都放在 `deploy` 阶段，生产部署自己失败时它根本不会跑（反而会被更早的 build/test 失败误触发）。把它挪到独立的 `rollback` 阶段，顺序才成立。如果前面阶段失败时不该动生产，再在脚本开头加一道判断，确认"当前线上的镜像确实不是期望版本"再执行 undo。

质量门禁前移：在 test 阶段接入 SAST、依赖漏洞扫描和代码规范检查，问题越早发现成本越低。敏感信息（registry 密码、kubeconfig）用 CI 的 Secret 变量注入，绝不写进 YAML。

流水线本身也要版本化——把 `.gitlab-ci.yml` 或 GitHub Actions 的 workflow 当作代码 review，避免有人在 UI 上偷偷改配置。

## 小结

一条可靠的 CI/CD 流水线 = 分阶段的质量门禁 + 可追溯的制品 + 可缓存的依赖 + 可控的发布策略 + 自动化的回滚。把"一次构建、逐级晋升"作为铁律，用金丝雀把变更风险限制在最小流量内，再用 `when: on_failure` 把回滚变成默认动作。做到这些，发布就从令人紧张的高风险操作，变成日常、无聊、可重复的流程——而"无聊"正是工程可靠性的最高赞誉。
