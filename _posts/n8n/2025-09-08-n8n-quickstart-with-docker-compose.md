---
title: "n8n 入门系列（一）：用 Docker Compose 快速搭建可用的 n8n 工作流环境"
date: 2025-09-08
last_modified_at: 2025-09-08
description: 面向小白的 n8n 入门：用 Docker Compose 10 分钟起一个可用的本地工作流环境，
categories:
  - N8N
tags:
  - n8n
  - AI工作流
  - 低代码
  - docker
  - docker-desktop
  - docker-compose
excerpt_separator: "<!--more-->"
---

> 面向小白的 n8n 入门：用 Docker Compose 10 分钟起一个可用的本地工作流环境。
{: .notice}
<!--more-->

# 1. Dify、Coze、n8n 的定位差异
- **n8n**：通用的工作流自动化平台，可自部署，长项是“连接万物 + 可视化编排 + 适度写代码（JS/表达式）”。近年加入了 AI/LLM 节点，但整体仍以系统/业务自动化为核心。
- **Coze**：字节旗下的 在线 Agent/Bot 平台，上手快、分发渠道全（各类社交/企业应用），同时提供了开源版（Coze Studio）便于企业私有化或二次开发。
- **Dify**：面向 AI 应用/Agent 的可视化编排平台，把模型管理、RAG Pipeline、观测与评估做在一起；自带知识库，但也能接外部 RAG（如 RAGFlow）。

# 2. 开始之前：准备好 Docker
若未安装，请先按我们的图文教程完成：
《Docker 安装与配置教程》：https://cryozerolabs.github.io/devops/install-and-configure-docker/

已安装的同学，打开 Docker Desktop，右上角鲸鱼图标为 Running 即可继续。
（可选自检：菜单 → Preferences/Settings → About 里能看到版本号；或终端运行 docker --version、docker compose version。）

> Windows 小贴士：建议启用 WSL2（Docker Desktop → Settings → Resources → WSL Integration 勾选你的发行版）。
{. :notice--primary}

# 3. 新建项目：n8n_compose

> 目标：你将得到一个项目文件夹（里面只有一个 docker-compose.yml）
{: .notice--primary}

## 3.1 Windows
1. 打开 文件资源管理器 → 进入 文档 (Documents)。
2. 建项目文件夹：右键空白处 → 新建 > 文件夹 → 命名为 n8n_compose。
3. <a href="/assets/2025/09/08/n8n-quickstart-with-docker-compose/docker-compose.yml" download>⬇️ 下载 docker-compose.yml</a> 并放在 `n8n_compose`目录中
4. 在 n8n_compose 文件夹空白处 Shift+右键 → 选择 在此处打开 PowerShell 窗口
![windows-open-termianl](/assets/2025/09/08/n8n-quickstart-with-docker-compose/windows-open-terminal.png)
5. 在终端里输入这唯一一条命令（回车）：
```
docker compose up -d
```

## 3.2 macOs
1. 打开 Finder → 进入 文稿 (Documents)。
2. 新建项目文件夹：`⌘+Shift+N` 新建文件夹，命名 `n8n_compose`。
3. <a href="/assets/2025/09/08/n8n-quickstart-with-docker-compose/docker-compose.yml" download>⬇️ 下载 docker-compose.yml</a> 并放在 `n8n_compose`目录中
4. Finder 里选中 n8n_compose → 右键 → 在终端中打开
![macos-open-termianl](/assets/2025/09/08/n8n-quickstart-with-docker-compose/macos-open-terminal.png)
5. 在终端里输入这唯一一条命令（回车）：
```
docker compose up -d
```

## 3.3 完整的docker-compose.yml

```yml
services:
  n8n:
    image: n8nio/n8n:1.111.0
    ports:
      - "5678:5678"
    environment:
      N8N_HOST: localhost
      N8N_PORT: 5678
      N8N_PROTOCOL: http
      WEBHOOK_URL: http://localhost:5678/
      DB_TYPE: sqlite
      DB_SQLITE_FILE: /home/node/.n8n/database.sqlite
      DB_SQLITE_POOL_SIZE: 1
      N8N_RUNNERS_ENABLED: "true"
      N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS: "true"
      N8N_SECURE_COOKIE: "false"
      N8N_BLOCK_ENV_ACCESS_IN_NODE: "false"
    volumes:
      - n8n_data:/home/node/.n8n
    restart: unless-stopped
volumes:
  n8n_data:
```

# 4. 查看n8n运行状态并访问n8n
前面执行完`docker compose up -d`之后，耐心等待下载。
![docker-compose-up-n8](/assets/2025/09/08/n8n-quickstart-with-docker-compose/docker-compose-up-n8n.png)

直到看见命令跑完
```shell
[+] Running 3/3
• Network n8n-compose_default   Created
• Volume "n8n-compose_nen_data" Created
• Container n8n-compose-n8n-1   Created
```

此时我们就可以在Docker Desktop中看到n8n服务的状态了。
![docker-compose-n8n-stack](/assets/2025/09/08/n8n-quickstart-with-docker-compose/docker-compose-n8n-stack.png)

接下来，只需要点击 “5678” 就会在浏览器中打开n8n。 

你也可以直接点击这里访问：[http://localhost:5678](http://localhost:5678)

# 5. n8n初始化向导

完成安装后，第一次访问 `http://localhost:5678` 会要求我们设置主账号信息，按照提示填写`邮箱`、`姓氏`、`名字`、`密码`。
> 注意, 密码长度要求 `8位` 及以上，并且至少包含`一个数字`、`一个大写字母`
{: .notice--warning}

![n8n-setup-step1](/assets/2025/09/08/n8n-quickstart-with-docker-compose/n8n-setup-step1.png)

接下来，按照实际情况填写一下问卷调查即可。
![n8n-setup-step2](/assets/2025/09/08/n8n-quickstart-with-docker-compose/n8n-setup-step2.png)

最后，你成功在本地完成了n8n的部署。
![n8n-home](/assets/2025/09/08/n8n-quickstart-with-docker-compose/n8n-home.png)

# 6. 后续访问
后续的访问，你只需要确保Docker Desktop启动。
接着在Docker Desktop中，查看n8n_compose的状态，`绿色`则是运行中。 

点击后面的"5678"即可使用默认浏览器打开n8n的web页面。 

或者直接访问：[http://localhost:5678](http://localhost:5678)
![docker-compose-n8n-stack](/assets/2025/09/08/n8n-quickstart-with-docker-compose/docker-compose-n8n-stack.png)

如果你发现状态图标是`灰色`的，那么点击后面的小三角，稍等片刻，直到上图的状态，即可按照上面的步骤访问n8n了。

![docker-compose-start-n8n](/assets/2025/09/08/n8n-quickstart-with-docker-compose/docker-compose-start-n8n.png)
