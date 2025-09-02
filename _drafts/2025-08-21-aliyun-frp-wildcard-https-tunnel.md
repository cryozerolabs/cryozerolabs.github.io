---
title: "用阿里云 + frp 搭建支持 HTTPS 的泛子域名内网穿透服务"
date: 2025-08-21
description: "基于阿里云 ECS 与 frp，自建支持 HTTPS 的泛子域名内网穿透服务。含域名解析、通配证书、Nginx 统一 TLS、权限加固与常见问题排查的完整实战指南"
categories:
  - devops
tags:
  - frp
  - 内网穿透
  - 阿里云
  - 泛子域名
  - 泛解析
  - Nginx
  - HTTPS
  - AliDNS
  - acme.sh
  - 反向代理
  - 通配证书
  - 微信回调内网穿透
---
# 1.背景与目标
- 什么是内网穿透 & 为何选 frp
- “泛子域名”能带来什么（多服务/多环境一键映射）
- 最终效果预览：app1.tunnel.example.com / dev-api.tunnel.example.com 即开即用

# 2.实现思路
- Ubuntu
- Nginx
- Docker
- 域名

# 3.部署 frps（服务端）
## 3.1 准备FRP服务端配置
```shell
sudo mkdir -p /opt/frps
```

## 3.2 用Docker跑frps
```shell
sudo docker run -d --name frps --restart unless-stopped \
  --network host \
  -v /opt/frps/frps.toml:/etc/frp/frps.toml:ro \
  fatedier/frps:v0.64.0 -c /etc/frp/frps.toml
```
> 本文编写与2025-08，fatedier/frps稳定版本为v0.64.0

## 3.3 Nginx配置
## 3.4 泛子域名与HTTPS


# 4.配置 frpc（客户端）

# 5.常见问题
## 