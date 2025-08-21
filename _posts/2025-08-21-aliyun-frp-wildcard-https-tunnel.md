---
title: "用阿里云 + frp 搭建支持 HTTPS 的泛子域名内网穿透服务"
date: 2025-08-21
description: "基于阿里云 ECS 与 frp，自建支持 HTTPS 的泛子域名内网穿透服务。含域名解析、通配证书、Nginx 统一 TLS、权限加固与常见问题排查的完整实战指南"
categories:
  - blog
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
# 0x00 背景与目标
- 什么是内网穿透 & 为何选 frp
- “泛子域名”能带来什么（多服务/多环境一键映射）
- 最终效果预览：app1.tunnel.example.com / dev-api.tunnel.example.com 即开即用

# 0x01 准备工作

# 0x02 部署 frps（服务端）
# 0x03 配置 frpc（客户端）

# 0x04 泛子域名与HTTPS


# 0x05 总结与后续