---
title: "TLS 证书管理与自动续期实践"
slug: "tls-cert-autorenew"
date: 2026-06-05
draft: false
description: "讲清证书链与 ACME 协议，落地 certbot 与 cert-manager 自动续期，并覆盖 HTTP-01/DNS-01 验证与过期监控。"
tags: ["TLS", "证书", "cert-manager", "HTTPS"]
categories: ["运维部署"]
---

## 证书过期：一种完全可以避免的故障

"网站突然全站不可信，浏览器一片红"——这类事故十有八九是证书过期。它不像宕机那样有渐变征兆，而是在某个整点准时爆发，影响所有用户。更尴尬的是，证书通常有效期 90 天，一年要续四五次，靠人记着手动续，漏掉只是时间问题。

证书管理的正确做法是把续期变成无人值守的自动化流程，并配上过期监控兜底。要落地自动化，先得理解证书链和 ACME 这两个基础概念，否则配起 certbot/cert-manager 只会知其然不知其所以然。

## 证书链与 ACME 协议

服务器下发的不是一张证书，而是一条证书链：叶证书（站点域名）→ 中间证书 → 根证书。浏览器本地信任根证书，通过逐级签名验证叶证书的可信性。配置时若只放叶证书、漏掉中间证书，就会出现"部分设备报不受信任"——因为有些客户端不会自行补全链条。`fullchain.pem` 就是叶证书与中间证书的拼接，应优先使用它。

Let's Encrypt 是免费 CA，通过 ACME 协议自动化签发与续期。ACME 的核心是"证明你控制这个域名"，主要有两种验证方式：

- **HTTP-01**：在域名的 `http://<domain>/.well-known/acme-challenge/<token>` 放置挑战文件，CA 通过 HTTP 访问验证。简单，但要求 80 端口可达，不支持泛域名。
- **DNS-01**：在域名下添加一条 `_acme-challenge` 的 TXT 记录。支持泛域名（`*.example.com`），不依赖 80 端口，但需要 DNS 服务商 API 凭据。

## certbot 自动续期与 cron

单机场景下 certbot 是最省事的方案。首次签发：

```bash
# 安装 certbot 与 Nginx 插件（Debian/Ubuntu）
apt-get install -y certbot python3-certbot-nginx

# 签发并自动改 Nginx 配置
certbot --nginx -d example.com -d www.example.com --email admin@example.com --agree-tos

# 干跑测试续期流程（不真正请求）
certbot renew --dry-run
```

certbot 安装时通常自带 systemd timer 或 cron 定时续期。若需自建，可写一条 cron 条目 `30 2 * * * /usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx"`，每天凌晨 2:30 检查，只有距到期不足 30 天才会真正续期。

`--post-hook` 确保续期后重载 Nginx 让新证书生效。certbot 的 `renew` 自带"未到期不请求"的逻辑，不必担心频繁调用触发限流。

## Kubernetes 中的 cert-manager

容器平台里更常用 cert-manager，它以 CRD 方式声明证书，自动完成签发与续期。

```yaml
# ClusterIssuer：定义签发机构（HTTP-01 方式）
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
# Certificate：声明要签发的域名，结果写入名为 web-tls 的 Secret
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: web-tls
  namespace: default
spec:
  secretName: web-tls
  dnsNames:
    - example.com
    - www.example.com
  issuerRef:
    name: letsencrypt-http
    kind: ClusterIssuer
---
# Ingress 只引用上面那个 Certificate 产出的 Secret，不再打 cluster-issuer 注解
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  tls:
    - hosts: [example.com, www.example.com]
      secretName: web-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

这里有个必须二选一的地方。用 `cert-manager.io/cluster-issuer` 注解，是让 ingress-shim **自动**为你生成一个 Certificate（域名取自 `spec.tls[].hosts`）；手写 Certificate 则是自己控制 `dnsNames`。**两种方式不能同时用**：注解会额外生成一个同名 Certificate，两个 Certificate 抢同一个 `secretName`，最终 Secret 里是哪张证书取决于谁的控制器后写入——`www.example.com` 就这么丢的，而且症状很隐蔽（主域名正常，只有带 www 的访问报证书不匹配）。

另外，`spec.tls[].hosts` 也要跟着写全。它决定 Ingress 认为这张证书该覆盖哪些域名，只写 `example.com` 的话，即使证书里有 www，Ingress 也不会为 www 的请求提供 TLS。

Nginx 侧使用证书：

```nginx
server {
    listen 443 ssl;
    http2 on;                     # nginx 1.25.1 起不再接受 listen ... http2 的写法
    server_name example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ...
}
```

`listen 443 ssl http2` 这种写法在 1.25.1 之前是标准姿势，之后被废弃（1.27 起只打警告不生效），新配置请把 `http2 on;` 单独写一行，否则你以为开了 HTTP/2，实际还在 1.1。

泛域名场景把 solver 改为 `dns01` 并配置对应服务商（如 route53、cloudflare）的凭据即可。

## 私钥保护与 HSTS

私钥的保护与证书本身同等重要。私钥文件应设为仅属主可读（`chmod 600`），绝不随代码仓库分发；一旦怀疑泄露，必须立即吊销并重新签发。关于吊销机制，这几年有个重要变化：**Let's Encrypt 已经停用 OCSP 响应服务，只提供 CRL**，CA/B Forum 也把 OCSP 改成了可选项，业界整体在往"短有效期 + CRL"走。这意味着吊销状态的分发比过去更慢、更依赖客户端主动查询——所以真正可靠的兜底不是吊销，而是**把证书寿命压短、把续期做快**，让泄露窗口本身就小到可接受。启用 HSTS（HTTP Strict Transport Security）能强制浏览器只走 HTTPS，防止首次访问被降级攻击；不过 HSTS 会被浏览器缓存较长时间，上线前务必确认全站 HTTPS 已就绪，否则回滚会非常棘手。把私钥权限、吊销预案与 HSTS 一起纳入证书规范，HTTPS 才算真正落地。

## 过期监控与最佳实践

自动化也可能失效（ACME 账户异常、DNS 凭据过期、验证路径被防火墙挡住）。必须加一道过期监控兜底，用黑盒导出器或脚本检查剩余天数：

```bash
# 检查证书是否会在 14 天内过期；交给 openssl 自己算，不依赖 GNU date
ENDPOINT=example.com:443
if echo | openssl s_client -connect $ENDPOINT -servername ${ENDPOINT%:*} 2>/dev/null \
     | openssl x509 -noout -checkend $((14 * 86400)) >/dev/null 2>&1; then
  echo "OK: $ENDPOINT 剩余有效期超过 14 天"
else
  echo "ALERT: $ENDPOINT 将在 14 天内过期，或握手失败，需人工介入"
fi
```

为什么不用 `date -d` 去解析 notAfter：`-d` 是 GNU coreutils 的扩展，macOS/BSD 自带的 `date` 没有这个参数，脚本在开发机上跑得好好的、换到 Mac 上就报 `illegal option -- d`。`openssl x509 -checkend <秒数>` 是 openssl 自带的能力，跨平台一致，也省掉了时区解析。如果要输出精确天数，用 `python3 -c` 解析 `-enddate` 更稳。

还有个细节：`-checkend` 在握手失败时同样返回非零，所以告警文案里要把"或握手失败"带上——否则证书没问题、只是网络不通，也会被当成"即将过期"，处理方向就跑偏了。

把这段做成定时任务并接入告警渠道，就能在续期失败时提前两周发现。

## 踩坑与最佳实践

始终用 `fullchain.pem` 而非单独的证书文件，漏配中间链是最常见的"部分客户端不信任"原因。

DNS-01 的服务商 API Token 权限要收敛到最小（仅可改 TXT 记录），并妥善存放为 Secret，避免泛域名证书私钥与 DNS 凭据同时泄露。

Let's Encrypt 有签发频率限制（同一域名每周张数上限），调试时务必用 staging 环境（`/directory` 换成 `acme-staging-v02...`），避免触发生产限流。

续期后一定要 reload/restart 服务端，否则内存里还是旧证书——这是"续了却没生效"的经典原因。

## 小结

TLS 证书管理的核心是"自动化 + 兜底"：用 ACME 协议配合 certbot 或 cert-manager 让签发与续期无人值守，用 HTTP-01 或 DNS-01 完成域名所有权验证，再用过期监控盯住自动化失效的边界。把证书当成会自己续期的“活”资源而不是一次性配置，配上 `fullchain`、最小权限凭据和 reload 钩子这些细节，你就能彻底告别证书过期的深夜告警。
