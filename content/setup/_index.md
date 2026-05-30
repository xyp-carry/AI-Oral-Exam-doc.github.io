---
title: 本地运行
weight: 30
bookToc: true
---

# 本地运行

当前后端使用 HTTPS 协议，因此本地调试时需要准备证书。证书生成成功后，可在项目中使用 HTTPS 启动服务。

## Windows 环境

可以在项目根目录执行以下命令生成本地证书并启动服务：

```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=localhost" && python -c "import uvicorn; uvicorn.run('main:app', host='0.0.0.0', port=443, ssl_keyfile='key.pem', ssl_certfile='cert.pem')"
```

## 说明

- `key.pem` 是本地私钥文件。
- `cert.pem` 是本地证书文件。
- `uvicorn.run` 会加载 `main:app` 并在 443 端口启动 HTTPS 服务。
- 如果本机 443 端口需要管理员权限或已被占用，可以在调试阶段改用其他端口。
