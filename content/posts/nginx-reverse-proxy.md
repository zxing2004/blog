---
title: "Nginx 反向代理配置避坑指南"
slug: "nginx-reverse-proxy"
date: 2026-04-02
draft: false
description: "深入讲解 Nginx 反向代理中 location 匹配、真实 IP 透传、WebSocket 升级、缓冲与超时连接池的高频坑点，并给出完整 server 块配置。"
tags: ["Nginx", "反向代理", "WebSocket", "运维"]
categories: ["运维部署"]
---

## 配置"看起来对"，线上却总出怪事

Nginx 的反向代理配置几乎每个后端服务都会写，但很多人的配置是从网上复制一段，改改端口就上线。结果应用中拿到的客户端 IP 全是 127.0.0.1，限流把所有用户当成同一个人；WebSocket 握手成功却几秒后断开；上传大文件偶发 502。这些问题往往不会在测试环境暴露，因为它们依赖特定的请求类型或流量形态。

反向代理的本质是 Nginx 代替客户端去访问后端，这中间会丢失或改写一部分原始信息。不把这些信息显式透传、不针对长连接和缓冲做调校，就会出现"代理层透明、应用层却蒙在鼓里"的割裂。下面按高频坑点逐一拆解。

## location 匹配优先级：不是谁在前谁说了算

location 的匹配和书写顺序基本无关，它更像三步判定，而不是一张优先级表：

1. 先找 `=` 精确匹配，命中就直接返回。
2. 再找**最长**的前缀匹配（普通前缀和 `^~` 一起比长度）。如果这个最长前缀带 `^~`，直接采用它，不再看正则。
3. 否则按配置里的书写顺序逐个试正则，第一个命中的就用；全都不命中，才回到第 2 步记下的最长前缀。

容易踩的坑是：写了 `location /api {}` 和 `location ~ \.php$ {}`，以为静态资源走前者、PHP 走后者，结果 `/api/user.php` 被正则截走。记住这条：**正则优先于普通前缀，唯一的例外是最长前缀带 `^~`**。注意是"最长前缀"带 `^~` 才算例外——`location ^~ /static/` 配上另一条更长的 `location /static/img/`，长的那条不带 `^~`，正则仍然有机会把请求截走。

## 透传真实 IP：X-Real-IP 与 X-Forwarded-For

经过反向代理后，后端看到的 TCP 对端是 Nginx 自己，因此需要 HTTP 头传递原始信息：

```nginx
location / {
    proxy_pass http://backend;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

`$proxy_add_x_forwarded_for` 会在原有 `X-Forwarded-For` 后追加 `$remote_addr`，形成链式记录；而 `$remote_addr` 只保留直连 Nginx 的那一跳。应用要取真实客户端时，应取 `X-Forwarded-For` 最左侧地址，且只在可信代理层之后才信任该头——否则攻击者可以伪造它。若 Nginx 前面还有 CDN/SLB，要用 `set_real_ip_from` 配合 `real_ip_header X-Forwarded-For` 还原真实来源。

## 一个完整的 server 块示范

把 WebSocket、超时、缓冲、连接池整合到一起：

```nginx
upstream backend {
    server 10.0.0.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:8080 max_fails=3 fail_timeout=30s;
    keepalive 64;                 # 到后端的长连接池
}

# Connection 头只有两种正确取值，且互斥：普通请求要空、WebSocket 要 upgrade。
# 用 map 按 $http_upgrade 自动切换，一条配置同时覆盖两种场景。
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      '';
}

server {
    listen 80;
    server_name example.com;

    # 允许的请求体上限
    client_max_body_size 100m;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;   # WebSocket 与 keepalive 都要求 1.1

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 升级：Upgrade 原样透传，Connection 由 map 决定
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # 超时：长连接场景要放宽
        proxy_connect_timeout 10s;
        proxy_send_timeout    300s;
        proxy_read_timeout    300s;
    }

    location /api/upload/ {
        proxy_pass http://backend;
        proxy_request_buffering off;   # 大文件流式上传，不先整包落盘
    }

    location /api/download/ {
        proxy_pass http://backend;
        proxy_buffering off;      # 大文件流式下载，避免内存/磁盘堆积
    }
}
```

WebSocket 需要 `Connection "upgrade"`，upstream keepalive 需要 `Connection ""`——这两个值互斥，写在同一个 location 里必然有一条被覆盖（`proxy_set_header` 同名头后写的那条生效）。正确做法就是上面这个 `map`：`$http_upgrade` 为空时映射成空串，有值时映射成 `upgrade`，一个变量同时满足两种场景。另一种等效做法是把 WebSocket 单独拆到一个 location。

## 缓冲与超时：看不见的性能开关

`proxy_buffering on`（默认）会让 Nginx 先把后端响应读入缓冲区再发给客户端，这对小响应有利，但会在下载大文件或 SSE 流式推送时引入延迟甚至内存压力。对 streaming 类接口应设 `proxy_buffering off`。

它管的是**响应**方向，别和管**请求**方向的 `proxy_request_buffering` 混了。后者默认也是 on：Nginx 会把客户端上传的请求体整包收完（超过内存阈值就落临时文件）再转给后端。几百 MB 的上传要关掉它，否则同一个文件先在你磁盘上写一遍、再读出来转发，白白多两倍 IO。

顺带说一句，`client_max_body_size` 只负责**限制**请求体大小，跟缓冲策略无关——想让它"不落盘"必须动 `proxy_request_buffering`，这是最常见的误解。

超时三兄弟含义不同：`proxy_connect_timeout` 是建连超时，`proxy_send_timeout` 是两次写操作间隔，`proxy_read_timeout` 是两次读操作间隔。WebSocket 断连多半是 `proxy_read_timeout`（默认 60s）太短，长连接无数据时被掐断，调到 300s 或配合业务心跳即可。

## 负载均衡与健康检查

upstream 的负载均衡策略也值得斟酌。默认轮询对多数场景够用，但实例性能不均时可用 `least_conn` 把新连接导向当前连接数最少的节点；需要会话保持的旧系统可用 `ip_hash`，但它容易因 NAT 把大量客户端聚成一团而导致负载倾斜，更稳妥的做法是把 session 外置到 Redis，从根上摆脱粘性会话。健康检查也要分清主动与被动：`max_fails` 与 `fail_timeout` 属于被动发现，故障实例仍会分到少量请求；严格的场景应结合服务发现机制主动摘除不健康节点，让流量只打在真正可用的后端上。

## 踩坑与最佳实践

`proxy_pass` 结尾是否带 `/` 会改变转发路径：`location /api/ { proxy_pass http://backend/; }` 会把 `/api/` 前缀替换掉，而末尾不加 `/` 则原样透传。这个差异常导致 404，务必对照 `nginx -T` 导出的完整配置检查。

`proxy_set_header` 具有继承性：若在 location 里写了任何一条，上层 server 的所有 `proxy_set_header` 都会被覆盖。建议把通用头集中放在 server 块，location 只写差异项。

改配置后先 `nginx -t` 校验语法，再 `nginx -s reload` 平滑重载，避免直接重启中断连接。

## 小结

Nginx 反向代理的坑，集中在"信息透传"与"连接行为"两条主线：用 `X-Forwarded-*` 系列头把客户端身份还原给后端，用 WebSocket 升级头、超时、缓冲与 keepalive 把连接行为调到贴合业务。把上面这份 server 块当作模板，再依据实际的 location 优先级和流式场景做裁剪，就能避开绝大多数线上怪事。代理层配置无小事，多一分显式声明，少十分深夜告警。
