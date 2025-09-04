---
title: "用 Nginx 通过自种 Cookie 做会话保持，像云厂商 SLB/CLB 的“植入 Cookie”那样复刻到自建 Nginx"
date: 2025-09-04
last_modified_at: 2025-09-04
description: 在 Ubuntu 上用 Nginx 通过自种 Cookie 做会话保持：WebSocket 路由用一致性哈希粘住，其他请求继续 least_conn。本文也会解释为什么不用 ip_hash，以及如何像云厂商 SLB/CLB 的“植入 Cookie”那样复刻到自建 Nginx。
categories:
  - devops
tags:
  - nginx
  - websocket
  - 负载均衡
  - 会话保持
  - sticky cookie
  - ubuntu
excerpt_separator: "<!--more-->"
---

在 Ubuntu 上用 Nginx 通过自种 Cookie 做会话保持：WebSocket 路由用一致性哈希粘住，其他请求继续 least_conn。本文也会解释为什么不用 ip_hash，以及如何像云厂商 SLB/CLB 的“植入 Cookie”那样复刻到自建 Nginx
{: .notice}

<!--more-->


# 1.场景与思路

- 诉求
  - WebSocket（或长连接、SignalR、SockJS 等）需要“粘”到同一后端。
  - 普通 HTTP 请求仍追求最少连接的公平分配（least_conn）。
- 做法
  - 只在需要“粘”的路由（如 /ws/）种一个黏性 Cookie（例如 SRV_STICKY），上游用 hash $cookie_SRV_STICKY consistent 做一致性哈希；
  - 其他路径继续用 least_conn。
- 为什么不用 ip_hash
  - IP 可能由 NAT/CDN/代理 共用，整团用户被“粘”到同一台，极易倾斜。
  - 移动网络 IP 频繁变化，粘性不稳定。
  - 你往往只想在部分路由粘住（如 /ws/），ip_hash 是上游级别策略，不够精细。
  - Cookie 粘性可 设置过期时间、按站点/路径生效、配合 SameSite/HttpOnly/Secure 更可控。

# 2.目录结构与约定
本文以 Ubuntu/Debian 常见布局为例：
```
/etc/nginx/nginx.conf
/etc/nginx/snippets/proxy_defaults.conf
/etc/nginx/snippets/ssl_params.conf
/etc/nginx/upstreams.d/app.conf
/etc/nginx/sites-available/example.conf
/etc/nginx/sites-enabled/example.conf -> ../sites-available/example.conf (软链)
```
> 文中所有 IP、域名、证书路径 均使用通用占位：
内网 IP：10.0.0.x，域名：www.example.com，证书：/etc/nginx/certs/www.example.com.{pem,key}
{: .notice--primary}

# 3.全局 nginx.conf（含“是否需要种 Cookie”的开关）
修改nginx.conf，增加 `$cookie_SRV_STICKY $need_sticky`, `$request_id $sticky_value` map组合。
```nginx
worker_processes  auto;

events {
    worker_connections  4096;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;
    server_tokens     off;

    # 统一日志
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    error_log   /var/log/nginx/error.log   warn;

    # WebSocket 连接升级的辅助变量
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    # 是否需要种黏性 Cookie（无 SRV_STICKY 时返回 1）
    map $cookie_SRV_STICKY $need_sticky {
        ""      1;   # 没有 -> 需要下发
        default 0;   # 已有 -> 不再下发
    }

    # 生成要下发的 Cookie 值（示例用 $request_id）
    map $request_id $sticky_value {
        default $request_id;
    }

    # 通用片段（代理默认头、强 TLS 参数等）
    include /etc/nginx/snippets/proxy_defaults.conf;
    include /etc/nginx/snippets/ssl_params.conf;

    # 上游池
    include /etc/nginx/upstreams.d/*.conf;

    # 站点
    include /etc/nginx/sites-enabled/*.conf;
}
```

> 这里的重点就在于map组合[`$cookie_SRV_STICKY $need_sticky`, `$request_id $sticky_value`], 让我们可以“只在需要的 location”里发 Cookie，其他位置不受影响。
{: .notice--primary}


# 4.upstream：普通请求 least_conn，WS 用 Cookie 一致性哈希
`/etc/nginx/upstreams.d/app.conf`：
```nginx
# 普通 HTTP 请求：最少连接
upstream app_http {
    least_conn;
    server 10.0.0.10:80 weight=2  max_fails=3 fail_timeout=10s;
    server 10.0.0.11:80 weight=1  max_fails=3 fail_timeout=10s;
    server 10.0.0.12:80 weight=1  max_fails=3 fail_timeout=10s;
    keepalive 64;
}

# WebSocket/长连接：按我们种的 Cookie 黏性（一致性哈希）
upstream app_ws_cookie {
    hash $cookie_SRV_STICKY consistent;
    server 10.0.0.10:80 weight=2 max_fails=3 fail_timeout=10s;
    server 10.0.0.11:80 weight=1 max_fails=3 fail_timeout=10s;
    server 10.0.0.12:80 weight=1 max_fails=3 fail_timeout=10s;
    keepalive 64;
}
```
> consistent 能减少后端节点增删时的“重映射”范围，连接更平滑。
{: .notice--primary}


# 5.站点 server：只在 /ws/ 下发 Cookie 并走黏性池
`/etc/nginx/sites-available/example.conf`:
```nginx
# HTTP -> HTTPS 跳转
server {
    listen      80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}

# HTTPS 主站点
server {
    listen      443 ssl;
     http2 on;
    server_name www.example.com;

    # 证书（fullchain）
    ssl_certificate     /etc/nginx/certs/www.example.com.pem;
    ssl_certificate_key /etc/nginx/certs/www.example.com.key;

    # 安全参数 & 通用反代头/WS
    include /etc/nginx/snippets/ssl_params.conf;
    include /etc/nginx/snippets/proxy_defaults.conf;

    # 普通请求：最少连接
    location / {
        proxy_pass http://app_http;
    }

    # WebSocket/SignalR 等：只在这里种 cookie，然后走 cookie 哈希上游
    location ^~ /ws/ {
        # 首次没 Cookie 才下发
        if ($need_sticky) {
            add_header Set-Cookie "SRV_STICKY=$sticky_value; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax" always;
        }

        proxy_pass http://app_ws_cookie;

        # WS 关键选项
        proxy_buffering   off;
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

`/etc/nginx/snippets/proxy_defaults.conf`（示例）：
```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# WebSocket 升级头
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection $connection_upgrade;

proxy_http_version 1.1;
```

> 为什么把“是否种 Cookie”的判断放在 location：
我们只在 /ws/ 下发 SRV_STICKY，保证普通请求不受影响，达到“按需粘住”。
{: .notice--primary}

# 6. 与 ip_hash 的对照

| 维度         | `ip_hash`                                          | Cookie 一致性哈希             |
| ------------ | -------------------------------------------------- | -------------------------------------------- |
| 精细度       | 只能对一个 upstream 生效，难以**按 location 控制** | 只在需要的路由种 Cookie，其它照常            |
| NAT/CDN 影响 | 多用户共享一个外网 IP 时会**挤到一台**             | 粘性以浏览器 Cookie 为准，更均衡             |
| 移动网络     | IP 频繁变化，粘性不稳定                            | Cookie 按过期时间可控                        |
| 安全/治理    | 无法设置过期、作用域                               | 支持 `Max-Age/Path/SameSite/HttpOnly/Secure` |
| 迁移/扩缩容  | 增删节点重映射不可控                               | `consistent` 降低抖动                        |

# 7.验证与排错
## 7.1 用 curl 看是否下发 Cookie
```bash
# 首次访问 WS 路由，应该看到 Set-Cookie: SRV_STICKY=...
curl -I https://www.example.com/ws/any
```

## 7.2 用浏览器开发者工具确认 SRV_STICKY
- 打开页面后按 F12（或右键→检查）。
  - Chrome/Edge：切到 Application → Storage → Cookies → 选择 https://www.example.com
  - Firefox：切到 Storage → Cookies → 选择对应站点
  -  Safari：Web Inspector → Storage → Cookies
- 确认存在名为 SRV_STICKY 的条目
- 若没有看到：
  - 先清理站点数据或开无痕窗口重试；
  - 确认访问的 URL 命中了 /ws/ 这个会下发 Cookie 的 location；
  - 看响应头里是否有 Set-Cookie: SRV_STICKY=...。

## 7.3 验证“粘性”是否生效
-  记下 SRV_STICKY 的值，多次访问 /ws/，观察是否命中同一后端
（可让后端在响应里加 X-Node: app-10.0.0.10 之类的标识，或查后端/ELB 日志）。
-  修改/删除浏览器里的 SRV_STICKY 值，再次连接应切到对应新节点。

## 7.4 在 Nginx 日志打印 Cookie（可选）
```nginx
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                'cookie=$cookie_SRV_STICKY';
```

# 8.生产化建议
- Cookie 设计：默认用 $request_id 简洁够用；也可改为用户 ID 的哈希（注意隐私与安全）。
- 过期时间：根据会话特性调整 Max-Age（如 30–120 分钟）。
- 灰度/回滚：将 /ws/ 的 upstream 独立成文件，方便切换策略或节点集。
- 健康检查：结合 max_fails/fail_timeout 或引入独立探针，确保异常节点迅速摘除。
- TLS：务必使用完整链证书（fullchain），并在 ssl_params.conf 固化安全套件与协议。

# 9.小结
- 用黏性 Cookie + 一致性哈希把 WebSocket 等需要会话保持的路由“粘住”，
- 其余请求继续走 least_conn，兼顾稳定性与吞吐均衡。
- 相比 ip_hash，Cookie 方案更精细、稳健、可治理，更贴近主流云负载均衡器的“植入 Cookie”做法。