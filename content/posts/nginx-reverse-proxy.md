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

location 的匹配有明确优先级，和书写顺序无关：

1. `=` 精确匹配（最高）
2. `^~` 前缀匹配，命中后不再检查正则
3. `~` 区分大小写的正则、`~*` 不区分大小写正则
4. 普通前缀匹配（无修饰符）
5. `/` 通用匹配（最低）

容易踩的坑是：写了 `location /api {}` 和 `location ~ \.php$ {}`，以为静态资源走前者、PHP 走后者，结果 `/api/user.php` 被正则截走。记住"正则优先于普通前缀，除非加 `^~`"这条铁律，就能解释大多数"路由不按预期走"的问题。

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

server {
    listen 80;
    server_name example.com;

    # 上传大文件时不缓冲整个请求体到磁盘
    client_max_body_size 100m;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;   # WebSocket 与 keepalive 都要求 1.1

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 升级必需的两个头
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 超时：长连接场景要放宽
        proxy_connect_timeout 10s;
        proxy_send_timeout    300s;
        proxy_read_timeout    300s;

        # 复用 upstream 长连接时清掉 Connection 头
        proxy_set_header Connection "";
    }

    location /api/download/ {
        proxy_pass http://backend;
        proxy_buffering off;      # 大文件流式下载，避免内存/磁盘堆积
    }
}
```

这里有两个看似矛盾的点：WebSocket 需要 `Connection "upgrade"`，而 upstream keepalive 需要 `Connection ""`。解决方式是把 keepalive 与 WebSocket 分到不同 location，或用 `map` 按 `$http_upgrade` 动态设置 `Connection` 值，避免全局写死。

## 缓冲与超时：看不见的性能开关

`proxy_buffering on`（默认）会让 Nginx 先把后端响应读入缓冲区再发给客户端，这对小响应有利，但会在下载大文件或 SSE 流式推送时引入延迟甚至内存压力。对 streaming 类接口应设 `proxy_buffering off`。

超时三兄弟含义不同：`proxy_connect_timeout` 是建连超时，`proxy_send_timeout` 是两次写操作间隔，`proxy_read_timeout` 是两次读操作间隔。WebSocket 断连多半是 `proxy_read_timeout`（默认 60s）太短，长连接无数据时被掐断，调到 300s 或配合业务心跳即可。

## 负载均衡与健康检查

upstream 的负载均衡策略也值得斟酌。默认轮询对多数场景够用，但实例性能不均时可用 `least_conn` 把新连接导向当前连接数最少的节点；需要会话保持的旧系统可用 `ip_hash`，但它容易因 NAT 把大量客户端聚成一团而导致负载倾斜，更稳妥的做法是把 session 外置到 Redis，从根上摆脱粘性会话。健康检查也要分清主动与被动：`max_fails` 与 `fail_timeout` 属于被动发现，故障实例仍会分到少量请求；严格的场景应结合服务发现机制主动摘除不健康节点，让流量只打在真正可用的后端上。

## 踩坑与最佳实践

`proxy_pass` 结尾是否带 `/` 会改变转发路径：`location /api/ { proxy_pass http://backend/; }` 会把 `/api/` 前缀替换掉，而末尾不加 `/` 则原样透传。这个差异常导致 404，务必对照 `nginx -T` 导出的完整配置检查。

`proxy_set_header` 具有继承性：若在 location 里写了任何一条，上层 server 的所有 `proxy_set_header` 都会被覆盖。建议把通用头集中放在 server 块，location 只写差异项。

改配置后先 `nginx -t` 校验语法，再 `nginx -s reload` 平滑重载，避免直接重启中断连接。

## 小结

Nginx 反向代理的坑，集中在"信息透传"与"连接行为"两条主线：用 `X-Forwarded-*` 系列头把客户端身份还原给后端，用 WebSocket 升级头、超时、缓冲与 keepalive 把连接行为调到贴合业务。把上面这份 server 块当作模板，再依据实际的 location 优先级和流式场景做裁剪，就能避开绝大多数线上怪事。代理层配置无小事，多一分显式声明，少十分深夜告警。
