---
title: "RESTful API 设计：容易被忽略的细节"
date: 2026-03-10
draft: false
description: "讨论 REST API 设计中常见但容易被忽视的问题，包括资源命名、状态码与版本管理。"
tags: ["API", "后端", "架构"]
categories: ["后端与架构"]
---

## 资源命名使用名词复数

URL 应表示资源，而非操作。推荐使用名词复数：

```
GET    /users          # 获取用户列表
GET    /users/123      # 获取单个用户
POST   /users          # 创建用户
PUT    /users/123      # 全量更新
PATCH  /users/123      # 部分更新
DELETE /users/123      # 删除用户
```

## 正确使用 HTTP 状态码

状态码是 API 语义的一部分，恰当地使用能让调用方更容易处理：

- `200 OK` —— 请求成功
- `201 Created` —— 资源创建成功，响应头应包含 `Location`
- `204 No Content` —— 成功但无响应体，常见于删除操作
- `400 Bad Request` —— 请求参数有误
- `401 Unauthorized` —— 未认证
- `403 Forbidden` —— 已认证但无权限
- `404 Not Found` —— 资源不存在
- `429 Too Many Requests` —— 触发限流
- `500 Internal Server Error` —— 服务端内部错误

## 版本管理

将版本放入 URL 路径是最直观的做法：

```
/api/v1/users
/api/v2/users
```

## 分页设计

对于列表接口，推荐统一的分页响应结构：

```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 135
  }
}
```

## 小结

良好的 API 设计应当**一致、可预测、自描述**。约定优于配置，一旦确立规范就应在整个系统内保持一致。
