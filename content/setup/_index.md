---
title: 本地运行
weight: 60
bookToc: false
---

# 本地运行

本地运行基于 Docker Compose 一键部署，一次性拉起前端（Node/Vite）、后端（Python/FastAPI）和 MySQL 三个服务。

## 前置条件

- 安装并启动 [Docker Desktop](https://www.docker.com/products/docker-desktop/)（Windows/macOS），或安装 Docker Engine 与 Compose 插件（Linux）。
- 仓库根目录自带 `.env` 配置文件，包含大模型等服务地址与密钥。如需使用自己的模型服务，编辑其中 `MODEL_URL`、`MODEL_API_KEY`、`DEEPSEEK_API_KEY`、`KIMI_API_KEY` 等字段。

## 一键部署

克隆仓库后，进入 `docker` 目录执行一条命令即可完成构建和启动：

```bash
git clone https://github.com/xyp-carry/AI-oral-exam.git
cd AI-oral-exam/docker
docker compose up -d --build
```

首次构建需要下载 PyTorch、npm 依赖等大量内容，耗时视网络情况可能较长；此后再次启动无需重新构建。首次发起口试时，容器内还会自动下载语音识别模型，同样存在一次性的等待。

启动过程中系统会自动完成数据库建库建表和自签名 HTTPS 证书生成，无需手工初始化。

## 访问入口

| 服务 | 地址 | 说明 |
|---|---|---|
| 前端 | http://localhost:5173 | Web 使用入口 |
| 后端 API | https://localhost:7860 | 使用自签名证书，浏览器提示不受信任时选择继续访问 |
| MySQL | localhost:3306 | 默认账号 root / 123456 |

## 常用运维命令

在 `docker` 目录执行：

```bash
docker compose ps              # 查看容器状态
docker compose logs -f backend # 跟踪后端日志
docker compose down            # 停止全部服务，数据保留
docker compose up -d           # 再次启动
docker compose up -d --build   # 代码更新后重新构建并启动
```

上传文件与考试记录持久化在仓库的 `updateFile/`、`exam_records/` 目录，语音模型缓存放于 Docker 卷，MySQL 数据保存在 `docker/mysql_data/` 目录，`docker compose down` 不会丢失这些数据。
