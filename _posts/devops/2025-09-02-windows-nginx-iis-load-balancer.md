---
title: "在 Windows 上部署 NGINX 作为 IIS 前置负载均衡（HTTPS/HTTP/2、WebSocket、服务化全攻略）"
date: 2025-09-02
last_modified_at: 2025-09-02
description: 在 Windows 上部署 NGINX 作为 IIS 前置负载均衡的实战指南：配置 HTTPS/HTTP/2 与 WebSocket，least_conn+权重调度，fullchain 证书与安全加固，基于粘性Cookie得会话保持，服务化运行与排错清单，附完整可复制示例。
categories:
  - devops
tags:
  - nginx
  - iis
  - reverse-proxy
  - websocket
  - https
  - windows-service
excerpt_separator: "<!--more-->"
---
在 Windows 上部署 NGINX 作为 IIS 前置负载均衡的实战指南：配置 HTTPS/HTTP/2 与 WebSocket，least_conn+权重调度，fullchain 证书与安全加固，基于粘性Cookie得会话保持，服务化运行与排错清单，附完整可复制示例。
{: .notice}

<!--more-->
# 1.架构与适用场景
将 TLS 终止、负载均衡与流量治理集中到 NGINX，IIS 只专注于应用本身。
- **证书集中**：TLS 只在 NGINX 终止，IIS 走内网纯 HTTP，维护成本更低；
- **流量治理**：支持负载均衡、灰度、蓝绿、压测分流；
- **跨技术栈**：无论后端是 IIS/.NET 还是其它服务，都能统一被接入（WebSocket 也 OK）。

> 说明：本文中的域名与 IP 统一使用示例值（如 `app.example.com`、`10.0.0.x`），请替换为你的真实信息。
{: .notice--primary}

# 2.准备与下载（Windows 稳定版）
- 前往 [https://nginx.org/en/download.html]() 下载 **Stable** 版本（稳定分支）。
- 解压到例如 `D:/nginx`
- 在服务器放通 **80/443**（入口）以及到后端 IIS 的 **8080/8081** 等内网端口（按你的配置）。

```
D:/nginx/
├─ certs/ # 证书（fullchain 与私钥）
├─ conf/
│ ├─ conf.d/ # 每个站点的 server 配置
│ ├─ snippets/ # 可复用片段（代理默认、TLS 参数等）
│ ├─ upstreams.d/ # 上游（负载均衡池）定义
│ └─ nginx.conf # 主配置
└─ logs/ # 访问与错误日志
```
> Windows 上 `nginx.conf` 默认在 `conf/` 目录中，`include snippets/*.conf` 等相对路径会以 `conf/` 为基准。
{: .notice--primary}

# 3.主配置 nginx.conf（含 WebSocket 与统一日志）
将以下内容保存为 `conf/nginx.conf`（核心与你提供的一致，已整理可直接用）：
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
    server_tokens off;

    # 统一日志（可按需分域名单独写）
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    error_log   logs/error.log   warn;

    # WebSocket 连接升级的辅助变量
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    # 通用片段（代理默认头、强TLS参数等）
    include snippets/proxy_defaults.conf;
    include snippets/ssl_params.conf;

    # 所有上游（负载均衡池）
    include upstreams.d/*.conf;

    # 每个站点/应用的 server 配置
    include conf.d/*.conf;
}
```

# 4.snippets 片段
在 `conf/snippets/` 下创建以下两个文件。

## 4.1 proxy_defaults.conf（反代默认头与 WS 支持）
``` nginx
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

# WebSocket 必需
proxy_set_header Upgrade           $http_upgrade;
proxy_set_header Connection        $connection_upgrade;

# 长连接/WS 建议拉长
proxy_read_timeout   3600s;
proxy_send_timeout   3600s;

# 上游失败切换策略（被动健康检查配合）
proxy_next_upstream error timeout http_500 http_502 http_503 http_504;

# 如后端对流式响应敏感，可视情况关闭缓冲
# proxy_buffering off;
```

## 4.2 ssl_params.conf（TLS 安全参数与响应头）
```
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!3DES:!ADH:!RC4:!DH:!DHE;
ssl_session_cache shared:SSL:20m;
ssl_session_timeout 10m;

# OCSP Stapling（需要 fullchain 及可用 DNS）
ssl_stapling on;
ssl_stapling_verify on;
resolver 223.5.5.5 223.6.6.6 114.114.114.114 119.29.29.29 valid=300s ipv6=off;
resolver_timeout 5s;

# 安全响应头（可按需调整）
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy strict-origin-when-cross-origin always;
```

> 证书务必使用 fullchain；如果中间证书缺失，客户端可能握手失败。
{: .notice--warning}

# 5.定义上游池 upstream（负载均衡）
在 `conf/upstreams.d/` 下创建 `app_upstream.conf`，按 CPU 核心数 经验分配权重，并启用最少连接法：
```nginx
# conf/upstreams.d/app_upstream.conf
upstream app_upstream {
    least_conn;  # 最少连接策略

    # 💡 请替换为你的真实内网 IP 与端口（示例注释中的“16/8核”为权重参考）
    server 10.0.0.2:8080 weight=2  max_fails=3 fail_timeout=10s;  # 16核
    server 10.0.0.3:8080 weight=1  max_fails=3 fail_timeout=10s;  # 8核
    server 10.0.0.4:8080 weight=1  max_fails=3 fail_timeout=10s;  # 8核

    keepalive 64;  # 复用到上游的长连接数量
}
```
> `max_fails/fail_timeout` 属于被动健康检查，当连接/响应失败累计到阈值后，暂时将该节点视为不可用；可结合 `proxy_next_upstream` 做快速切换。
{: .notice--primary}

# 6.站点 server 配置（HTTP→HTTPS、HTTP/2、证书路径）
在 `conf/conf.d/` 下创建 `app.example.com.conf`：
```nginx
# conf/conf.d/app.example.com.conf

# 80 端口：强制跳转到 HTTPS
server {
    listen      80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}

# 443 端口：主站（启用 HTTP/2）
server {
    listen      443 ssl;
    http2       on;
    server_name app.example.com;

    # 证书路径（fullchain + 私钥），请替换为你的真实文件位置
    ssl_certificate      D:/nginx/certs/app.example.com.pem;
    ssl_certificate_key  D:/nginx/certs/app.example.com.key;

    # 引用 TLS 安全参数 & 通用反代设置
    include snippets/ssl_params.conf;
    include snippets/proxy_defaults.conf;

    # 应用转发（根路径）
    location / {
        proxy_pass http://app_upstream;
    }

    # 如需限制上传大小（示例 50M）：
    # client_max_body_size 50m;
}
```

# 7.IIS 侧设置
## IIS 操作：
- 站点 A：HTTP 绑定 `*:8080`，主机名留空；
- 站点 B：HTTP 绑定 `*:8081`，主机名留空；
- 其它站点以此类推（8082、8083…）。
## NGINX 配合（示例：两个域名分别指向不同端口站点）：

```nginx
# conf/upstreams.d/appA_upstream.conf
upstream appA_upstream {
    least_conn;
    server 10.0.0.2:8080 weight=2 max_fails=3 fail_timeout=10s;
    server 10.0.0.3:8080 weight=1 max_fails=3 fail_timeout=10s;
    keepalive 64;
}

# conf/upstreams.d/appB_upstream.conf
upstream appB_upstream {
    least_conn;
    server 10.0.0.2:8081 weight=2 max_fails=3 fail_timeout=10s;
    server 10.0.0.4:8081 weight=1 max_fails=3 fail_timeout=10s;
    keepalive 64;
}

# conf/conf.d/app.example.com.conf
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    http2  on;
    server_name app.example.com;
    ssl_certificate     D:/nginx/certs/app.example.com.pem;
    ssl_certificate_key D:/nginx/certs/app.example.com.key;
    include snippets/ssl_params.conf;
    include snippets/proxy_defaults.conf;
    location / { proxy_pass http://appA_upstream; }
}

# conf/conf.d/admin.example.com.conf
server {
    listen 80;
    server_name admin.example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    http2  on;
    server_name admin.example.com;
    ssl_certificate     D:/nginx/certs/admin.example.com.pem;
    ssl_certificate_key D:/nginx/certs/admin.example.com.key;
    include snippets/ssl_params.conf;
    include snippets/proxy_defaults.conf;
    location / { proxy_pass http://appB_upstream; }
}
```

# 8.将 NGINX 以 Windows 服务运行（服务化）
Windows 版本的 nginx 被视为测试版本，截止至`1.29.1`，nginx作为服务运行，仍然是future。
幸运的是，我找到了一个开源的解决方案 [https://github.com/winsw/winsw]()，它是用.net构建的，允许将任何可执行文件作为服务运行。

winsw允许将任何.exe 文件作为 Windows 服务使用。它使用 XML 来处理服务的配置和安装。
对于Windows x64可以直接下载

[https://github.com/winsw/winsw/releases/download/v2.12.0/WinSW-x64.exe]()

下载后，进入nginx目录，并将文件复制到目录下，在这个例子中是 `D:/nginx`，并将其重命名为 `nginx-service.exe`。

接下来，创建一个 nginx-service.xml 来描述服务，注意调整nginx路径：
```xml
<service>
  <id>nginx</id>
  <name>Nginx</name>
  <description>Nginx web server</description>

  <executable>D:\nginx\nginx.exe</executable>
  <workingdirectory>D:\nginx</workingdirectory>

  <!-- 停服务时优雅退出 -->
  <stopexecutable>D:\nginx\nginx.exe</stopexecutable>
  <stoparguments>-s quit</stoparguments>

  <startmode>Automatic</startmode>

  <logpath>D:\nginx\logs</logpath>
  <log mode="roll-by-size">
    <sizeThreshold>10485760</sizeThreshold>
    <keepFiles>5</keepFiles>
  </log>
</service>
```

以管理员权限打开 PowerShell 或 CMD：
```powershell
# 安装为服务（nginx-service.exe）
.\nginx-service.exe install

# 启动服务
.\nginx-service.exe start

# 停止服务
.\nginx-service.exe stop
# 重启服务
.\nginx-service.exe stop
# 卸载服务
.\nginx-service.exe uninstall
```

nginx自检仍然使用 `.\nginx.exe -t`

如果出现意外，比如自己用.\nginx.exe 启动了nginx 而导致脱离了服务的控制，无法通过nginx-service.exe对服务进行重启的，就需要强制结束了。
所以，不要自己的用.\nginx启动，要重启之前，先自检，因为重启报错也可能导致脱离服务管控。
```shell
taskkill /F /IM nginx.exe

```

> 注意：
> - 确认进程权限与工作目录正确，否则热加载/服务管理可能失败；
> - 路径含空格时，建议简化安装路径或使用引号。
{: .notice--warning}

# 9.（补充）实现基于粘性Cookie的会话保持
前面配置的upstream，使用least_conn更均衡的分配。但是用到websocket就不行了。
这里我参考了阿里云负载均衡服务（SLB）中CLB的 植入Cookie方式，来保持会话粘性。
## 9.1 修改nginx.conf
在 `http {}` 片段中插入代码，其中`SRV_STICKY`就是我们要植入的cookie，记住等会会用到，当然也可以根据实际情况修改。
```nginx
	# 是否需要种黏性Cookie
	map $cookie_SRV_STICKY $need_sticky {
		""      1;   # 没有 -> 需要下发
		default 0;
	}

	# 生成要下发的Cookie值（只在 need_sticky=1 时使用）
	map $request_id $sticky_value {
		default $request_id;
	}
```

## 9.3 增加upstream以支持sticky
修改`conf/upstreams.d/app_upstream.conf` 增加app_signalr_cookie
```nginx
# conf/upstreams.d/app_upstream.conf
upstream app_upstream {
    ... 原来的不要动
}

# 新增的
upstream app_signalr_cookie {
    hash $cookie_SRV_STICKY consistent;  # 按我们种的 cookie 黏性

    # 💡 请替换为你的真实内网 IP 与端口（示例注释中的“16/8核”为权重参考）
    server 10.0.0.2:8080 weight=2  max_fails=3 fail_timeout=10s;  # 16核
    server 10.0.0.3:8080 weight=1  max_fails=3 fail_timeout=10s;  # 8核
    server 10.0.0.4:8080 weight=1  max_fails=3 fail_timeout=10s;  # 8核

    keepalive 64;  # 复用到上游的长连接数量
}
```

## 9.2 对指定的websocket路径设置粘性会话
修改`conf/conf.d/app.example.com.conf`， 增加一条规则。
```nginx
location ^~ /ws/ {
		# 第一次没 Cookie 才种
		if ($need_sticky) {
			add_header Set-Cookie "SRV_STICKY=$sticky_value; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax" always;
		}

		proxy_pass http://app_signalr_cookie;
		proxy_buffering off;
		proxy_read_timeout  3600s;
		proxy_send_timeout  3600s;
	}
```

接下来
1. 重启nginx
2. 在浏览器中验证Cookie中是否有key：`SRV_STICKY`

# 10.常见问题与排错清单
- 502/504：上游不可达或超时。检查：
  - 内网防火墙是否放通 8080/8081；
  - IIS 站点是否启动、应用池是否健康；
  - `max_fails/fail_timeout` 是否过于严格；
  - `proxy_read_timeout/proxy_send_timeout` 是否偏短。
- 413（Payload Too Large）：增加 client_max_body_size（例如 50m）。
- 无法握手/证书错误：确保证书为 fullchain，密钥与证书匹配，域名一致。
- WebSocket 400/426：确认已设置 `proxy_http_version 1.1`、`Upgrade/Connection` 头与 map $http_upgrade。
- 真实客户端 IP：应用/日志读取 `X-Forwarded-For；IIS` 可在高级日志字段中记录该头。
- 日志定位：查看 `logs/error.log` 与 `logs/access.log`，快速筛选关键字符串或状态码。
- 端口占用：Windows 上用 `netstat -ano | findstr :80 / :443 / :8080` 定位占用进程。

# 11.发布前检查清单（Checklist）
- ✅ 域名解析正确，公网仅指向 NGINX 所在机器；
- ✅ 证书（fullchain/privkey）路径与权限无误；
- ✅ Windows 防火墙、云安全组等放通 80/443（入口）与 8080/8081（到 IIS）；
- ✅ NGINX 配置 `nginx -t` 自检通过，`-s reload` 生效；
- ✅ 访问 https://app.example.com/ 正常；后端 8080/8081 健康；
- ✅ WebSocket/长连接业务已验证，超时时间设置合理；
- ✅ 日志中出现 200/301/101 等预期状态码，后端无异常 5xx。
