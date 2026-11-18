---
title: "基础设施即代码：Terraform 入门与状态管理"
slug: "terraform-iac-basics"
date: 2026-07-17
draft: false
description: "从声明式 IaC 理念到 HCL、provider、resource，讲透 Terraform 的 state 与远程后端、plan/apply 流程和漂移检测。"
tags: ["Terraform", "IaC", "HCL", "DevOps"]
categories: ["运维部署"]
---

## 从"点鼠标建资源"到"代码定义基础设施"

早期搭环境靠控制台点鼠标，或者写一堆零散脚本。结果是环境之间不一致、变更没有记录、想复刻一套预发环境得翻聊天记录找当时点了哪些选项。这就是所谓的"雪花服务器"——每台都独一无二，没人敢动。

基础设施即代码（IaC）把基础设施的定义写进版本库：想要什么资源、什么配置，全部声明在代码里，任何人执行同一份代码都能得到一致的环境。Terraform 是其中最主流的工具，它用声明式语言 HCL 描述目标状态，自动计算出"从现状到目标需要做什么"，再按依赖顺序执行。理解"声明式"是理解 Terraform 的钥匙：你描述的是**结果**，而不是**步骤**。

## HCL、provider 与 resource

HCL（HashiCorp Configuration Language）的核心是两类块：`provider` 定义要去哪个平台操作，`resource` 定义要创建的资源。以在云上创建一台虚拟机和安全组为例：

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_security_group" "web" {
  name = "web-sg"
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.micro"
  security_groups = [aws_security_group.web.name]

  tags = {
    Name = "web-server"
    Env  = "prod"
  }
}
```

`security_groups = [aws_security_group.web.name]` 体现了资源间引用：Terraform 据此建立依赖图，保证先建安全组再建虚拟机。这种隐式依赖是声明式语言的精髓——你只说"引用它"，调度顺序由工具推导。

## state：Terraform 的记忆与远程后端

Terraform 靠一个 state 文件记录"代码里声明的资源"与"真实世界资源"的映射。没有 state，它就不知道某个资源是否已经存在、该更新还是该新建。因此 state 是 Terraform 最关键的资产，必须妥善保管与共享。

单机本地 state 适合尝鲜，团队协作必须用远程后端，配合加锁防止并发冲突。以 S3 + DynamoDB 锁为例：

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/network/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-locks"   # 状态锁，防止两人同时 apply
    encrypt        = true                # 静态加密
  }
}
```

## plan / apply 与漂移检测

初始化后端后即可进入标准流程——"先看后改"：

```bash
terraform init        # 下载 provider、初始化后端
terraform fmt         # 格式化 HCL
terraform validate    # 语法与引用校验

# 生成执行计划，预览将新增/修改/销毁哪些资源
terraform plan -out=tfplan

# 确认无误后应用；生产环境建议由 CI 统一执行
terraform apply tfplan

# 销毁（高危，务必先 plan 确认）
terraform destroy -plan
```

`plan` 是安全阀门——任何变更先看到 diffs 再决定。`apply` 成功后 state 被更新。

漂移检测用于发现"代码没变，但线上被人手动改过"的情况：定期跑 `terraform plan -detailed-exitcode`，若返回码为 2 表示存在差异（漂移），可据此告警。这是防止控制台"手动改动"破坏 IaC 一致性的有效手段。

## 模块复用

当多个环境需要相同的一套资源时，用 module 封装。模块就是一个含 `.tf` 文件的目录，通过 `variable` 接收参数、`output` 暴露结果：

```hcl
module "web_cluster" {
  source        = "./modules/web"
  instance_type = "t3.small"
  env           = "staging"
}
```

把网络、安全组等通用能力做成模块，各环境只传入差异参数，既减少重复又保证基线一致。

## 多环境与凭证管理

多环境管理还可用 workspace 或目录隔离。workspace 适合用同一套代码管理多个相似环境（dev/staging/prod），各环境持有独立的 state；目录隔离更直观，每个环境一个目录、各自引用远程后端的不同 key。小团队用 workspace 简单，大团队倾向目录隔离以配合不同的权限边界。CI 中运行 Terraform 时，云凭证应通过 OIDC 短期授权注入，而不是把长期 Access Key 写死在流水线里。每次 plan 的输出建议归档留存——谁在什么时间发起了什么变更，都要能追溯，这既是审计要求，也是出问题时快速定位的依据。

## 踩坑与最佳实践

state 含敏感信息（资源 ID、甚至明文密码），远程后端必须开启加密与最小权限访问，且绝不能提交进 Git。

不要在多人之间共享本地 state 文件，也不要手动编辑 state；真要迁移资源用 `terraform state mv`，导入已有资源用 `terraform import`。

`version = "~> 5.0"` 这类约束可锁定大版本，避免 provider 升级带来破坏性变更；升级前先读 CHANGELOG。

`terraform destroy` 不可逆，生产环境应通过 CI 的权限控制和审批来约束，而非人人可执行。

## 小结

Terraform 的价值不在"自动建资源"，而在把基础设施变成可审查、可复现、可回滚的代码。入门的关键是三件事：用 HCL 声明式描述目标资源与依赖，理解 state 的核心地位并用远程后端 + 锁妥善管理，养成 `plan` 先行、模块复用、漂移检测的习惯。当环境可以用一次 `apply` 精确复现时，你才算真正把基础设施握在了手里，而不是供在控制台里。
