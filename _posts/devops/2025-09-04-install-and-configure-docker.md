---
title: "一次搞定 Docker 安装与配置（含代理与镜像加速）"
date: 2025-09-04
last_modified_at: 2025-09-04
description: 这篇指南带你在 macOS、Windows、Ubuntu 上安装 Docker，并手把手配置镜像加速与网络代理，最后用 hello-world 验证一切正常。
categories:
  - devops
tags:
  - docker
  - docker-desktop
  - ubuntu
  - registry-mirrors
  - proxy
excerpt_separator: "<!--more-->"
---

这篇指南带你在 macOS、Windows、Ubuntu 上安装 Docker，并手把手配置镜像加速与网络代理，最后用 hello-world 验证一切正常。
{: .notice}
<!--more-->

# 1. macOS / Windows: 安装 Docker Desktop
## 1.1 下载安装 Docker Desktop
1. 访问官网下载安装包，按照向导安装。
2. Windows 建议启用 WSL 2 后端（安装向导会引导完成）。

## 1.2 快速自检
在Docker Desktop中，打开Terminal，执行以下命令：
```bash
docker run --rm hello-world
```
![docker-open-termianl](/assets/images/2025/09/04/docker-open-termianl.png)

若 `hello-world` 成功输出欢迎文案，说明安装 OK。若失败，继续阅读第 [3.配置镜像加速registry-mirror](#3配置镜像加速registry-mirrors) 后再试。


# 2.Ubuntu: 官方仓库安装
# 2.1 添加 Docker 官方源并安装
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 放置 keyring
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 添加 repo（注意使用你当前系统的代号）
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> 可选：把当前用户加入 `docker` 组（避免每次 `sudo`）
{: .notice--primary}

```bash
sudo usermod -aG docker $USER
# 重新登录终端后生效
```

# 2.2 快速自检
```
docker version
docker info
docker run --rm hello-world
```
若 `hello-world` 成功输出欢迎文案（参考 [1.2-快速自检](#12-快速自检) )，说明安装 OK。若失败，继续阅读 [3.配置镜像加速registry-mirror](#3配置镜像加速registry-mirrors) 再试。


# 3.配置镜像加速（registry-mirrors）
## 3.1 为什么要配 registry-mirrors？
网络连通与跨境链路：直连 registry-1.docker.io 常遇到链路拥塞、解析与互联互通质量不佳，导致 docker pull 慢或超时。

但历经多轮整改，若干高校/公共镜像已停服或限流，云厂商的“官方加速器”也有策略调整（例如阿里云 ACR 公共加速器停止同步最新镜像，需改用订阅/全球加速等方案）。所以地址的可用性具有时效性，建议准备多个备选，或考虑自建代理。
- [阿里云官方镜像加速: ACR镜像加速目前已停止同步最新镜像](https://help.aliyun.com/zh/acr/user-guide/accelerate-the-pulls-of-docker-official-images)
- [中科大镜像: 所有镜像缓存已暂停服务](https://mirrors.ustc.edu.cn/help/dockerhub.html)

官方也支持通过 --registry-mirror 启动参数设置，但落盘到 daemon.json 可持久化管理。

> 以下镜像加速地址更新时间为 `2025-09-04` 如需测试是否可用，可以直接运行命令 `docker pull run-docker.cn/hello-world` 进行测试，其中`run-docker.cn`换成需要测试的镜像加速器的域名
{: .notice--danger}

## 3.1 Docker Desktop(macOS 和 Windows): 两步就够
打开 Docker Desktop → Settings → Docker Engine，在 JSON 中添加/合并：
```json
{
  "registry-mirrors": [
    "http://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "http://run-docker.cn",
    "http://docker.hlmirror.com",
    "http://docker.tbedu.top",
    "https://dhub.kubesre.xyz"
  ]
}
```
![docker-open-termianl](/assets/images/2025/09/04/docker-engine-set-registry-mirrors.png)
点击 `Apply & restart` 使其生效（不同版本也可在 Settings 总页签里调整，或直接改 settings-store.json）。

## 3.2 适用于Ubuntu(Linux): 编辑 daemon.json
### 3.2.1 新建或编辑 /etc/docker/daemon.json：
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "registry-mirrors": [
    "http://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "http://run-docker.cn",
    "http://docker.hlmirror.com",
    "http://docker.tbedu.top",
    "https://dhub.kubesre.xyz"
  ]
}
EOF
```
### 3.2.2 重载并重启
```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 3.3 使用 hello-world 验证安装与网络
打开终端（或 PowerShell）执行：
```bash
# 1) 先看镜像加速是否生效
docker info | grep -A3 'Registry Mirrors'
# 2) 拉取镜像测试
docker run --rm hello-world
```


> 到这里，你已经完成了 Docker 的安装、镜像加速与网络代理配置，并用 hello-world 验证通过
{: .notice--primary}