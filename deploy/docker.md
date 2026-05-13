# Docker 部署指南 (nikolaik/python-nodejs)

> 适用环境：Docker / nikolaik/python-nodejs 镜像 / 容器化 Hermes Agent

## 1. 系统要求

- **Docker**: 20.10+
- **镜像**: `nikolaik/python-nodejs:latest` 或指定标签
- **主机**: 任意支持 Docker 的平台（Linux / macOS / Windows）

## 2. 镜像说明

`nikolaik/python-nodejs` 镜像同时包含 Python 和 Node.js 运行时，适合 Hermes Agent 所需的多语言环境。

### 可用的标签

```bash
# 查看可用的 Python + Node.js 组合
docker search nikolaik/python-nodejs

# 推荐使用
nikolaik/python-nodejs:latest         # Python 最新 + Node LTS
nikolaik/python-nodejs:python3.12-node22  # 指定版本
nikolaik/python-nodejs:python3.11-node20
```

### 拉取镜像

```bash
docker pull nikolaik/python-nodejs:latest
```

## 3. 目录结构

在主机上创建以下目录结构：

```
~/hermes-docker/
├── data/              # Hermes 持久化数据
├── config/            # 配置文件
├── logs/              # 日志输出
├── .env               # 环境变量
├── docker-compose.yml # 推荐方式
└── Dockerfile         # 自定义构建（可选）
```

创建目录：

```bash
mkdir -p ~/hermes-docker/{data,config,logs}
```

## 4. Docker Compose 部署（推荐）

### docker-compose.yml

```yaml
version: "3.9"

services:
  hermes:
    image: nikolaik/python-nodejs:latest
    container_name: hermes-agent
    hostname: hermes-docker
    restart: unless-stopped

    ports:
      # Hermes 服务端口（根据实际需求调整）
      - "8080:8080"

    volumes:
      # 持久化数据
      - ./data:/home/hermes/data
      - ./config:/home/hermes/.config/hermes
      - ./logs:/home/hermes/logs

      # 可选：挂载 Obsidian vault
      # - ~/obsidian-vault:/home/hermes/vault

    environment:
      # Hermes 配置
      - HERMES_CONFIG_DIR=/home/hermes/.config/hermes
      - HERMES_DATA_DIR=/home/hermes/data
      - HERMES_LOG_DIR=/home/hermes/logs

      # 代理设置（如需）
      # - HTTP_PROXY=http://host.docker.internal:7890
      # - HTTPS_PROXY=http://host.docker.internal:7890
      # - NO_PROXY=localhost,127.0.0.1

      # 时区
      - TZ=Asia/Shanghai

    working_dir: /home/hermes

    # 健康检查
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s

    # 资源限制（根据主机调整）
    deploy:
      resources:
        limits:
          cpus: "4"
          memory: "8G"
        reservations:
          cpus: "1"
          memory: "2G"

    # 日志配置
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  default:
    name: hermes-network
```

### 启动服务

```bash
cd ~/hermes-docker
docker compose up -d

# 查看日志
docker compose logs -f

# 查看状态
docker compose ps
```

## 5. 纯 Docker CLI 部署

如不使用 Compose，可直接运行：

```bash
docker run -d \
  --name hermes-agent \
  --restart unless-stopped \
  -p 8080:8080 \
  -v $(pwd)/data:/home/hermes/data \
  -v $(pwd)/config:/home/hermes/.config/hermes \
  -v $(pwd)/logs:/home/hermes/logs \
  -e HERMES_CONFIG_DIR=/home/hermes/.config/hermes \
  -e HERMES_DATA_DIR=/home/hermes/data \
  -e TZ=Asia/Shanghai \
  --cpus="4" \
  --memory="8g" \
  nikolaik/python-nodejs:latest \
  hermes --serve
```

## 6. 镜像内安装 Hermes Agent

### 6.1 在容器内安装

```bash
# 进入容器
docker exec -it hermes-agent bash

# 在容器内
cd /home/hermes
git clone https://github.com/NousResearch/hermes-agent
cd hermes-agent
# 按官方文档安装
```

### 6.2 构建自定义镜像（推荐生产环境）

创建 `Dockerfile`：

```dockerfile
FROM nikolaik/python-nodejs:latest

# 创建非 root 用户
RUN groupadd -r hermes && useradd -r -g hermes -m -d /home/hermes hermes

# 安装系统依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 安装 Hermes Agent
RUN git clone https://github.com/NousResearch/hermes-agent /home/hermes/hermes-agent
WORKDIR /home/hermes/hermes-agent
# RUN pip install -r requirements.txt  # 以实际文档为准
# RUN npm install                      # 以实际文档为准

# 创建必要目录
RUN mkdir -p /home/hermes/data /home/hermes/logs /home/hermes/.config/hermes

# 权限
RUN chown -R hermes:hermes /home/hermes

USER hermes
WORKDIR /home/hermes

# 暴露端口
EXPOSE 8080

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["hermes", "--serve"]
```

构建并运行：

```bash
docker build -t hermes-agent-custom .
docker run -d --name hermes-custom --restart unless-stopped \
  -p 8080:8080 \
  -v ~/hermes-docker/data:/home/hermes/data \
  -v ~/hermes-docker/config:/home/hermes/.config/hermes \
  -v ~/hermes-docker/logs:/home/hermes/logs \
  hermes-agent-custom
```

## 7. 管理与运维

### 常用命令

```bash
# 查看日志
docker logs -f hermes-agent

# 进入容器
docker exec -it hermes-agent bash

# 重启
docker restart hermes-agent

# 停止
docker stop hermes-agent

# 删除容器（数据保存在卷中不受影响）
docker rm hermes-agent
```

### 更新流程

```bash
cd ~/hermes-docker

# 拉取最新镜像
docker pull nikolaik/python-nodejs:latest

# 重建并重启
docker compose down
docker compose up -d
```

### 资源监控

```bash
# 查看容器资源占用
docker stats hermes-agent

# 查看详细资源使用
docker inspect hermes-agent | jq '.[].HostConfig.Resources'
```

## 8. 网络配置

### 主机网络模式（如有特殊网络需求）

```yaml
# docker-compose.yml 中添加
network_mode: "host"
```

### 使用 Docker 内部网络

```yaml
networks:
  hermes-network:
    driver: bridge
```

其他容器可通过 `hermes-agent:8080` 访问 Hermes。

## 9. 与多设备协同

Docker 部署适合在服务器上运行，可与 Linux/Mac/Android 端配合：

```
Docker 服务器 (云/家庭服务器)
       │
       ├── 暴露 8080 端口供外部访问
       ├── 通过 Git 同步配置到各端
       └── 统一数据管理
```

## 10. 故障排查

| 问题 | 可能原因 | 解决办法 |
|------|---------|---------|
| 容器无法启动 | 端口冲突 | 修改外部映射端口 `8080:8080` → `9090:8080` |
| 权限错误 | 卷挂载权限 | `chown -R 1000:1000 ~/hermes-docker/data` |
| 内存不足 | 资源限制过低 | 调高 `--memory` / `deploy.resources.limits.memory` |
| 网络超时 | 代理未配置 | 添加 `HTTP_PROXY` 环境变量 |
| 镜像拉取慢 | 网络问题 | 配置 Docker 镜像加速器 |

### Docker 镜像加速器配置

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

重启 Docker 生效：

```bash
sudo systemctl restart docker
```

---

*最后更新: 2026-05-14*
