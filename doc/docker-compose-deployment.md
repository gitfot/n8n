# n8n Docker Compose 部署指南（从源码构建）

本文档介绍如何从源码构建 n8n Docker 镜像，并使用 Docker Compose 部署到服务器。

## 目录

- [前置要求](#前置要求)
- [项目构建流程](#项目构建流程)
- [Docker 镜像构建](#docker-镜像构建)
- [Docker Compose 部署](#docker-compose-部署)
  - [方案一：SQLite（轻量部署）](#方案一sqlite轻量部署)
  - [方案二：PostgreSQL（生产推荐）](#方案二postgresql生产推荐)
  - [方案三：PostgreSQL + Task Runners（完整部署）](#方案三postgresql--task-runners完整部署)
- [环境变量说明](#环境变量说明)
- [数据持久化](#数据持久化)
- [反向代理配置](#反向代理配置)
- [维护与更新](#维护与更新)
- [常见问题](#常见问题)

---

## 前置要求

服务器环境：

- Docker >= 20.10
- Docker Compose >= 2.0（`docker compose` 命令）
- 至少 2GB 内存

构建环境（可以是本地开发机或 CI）：

- Node.js >= 22.16
- pnpm >= 10.22.0
- Docker（用于构建镜像）

## 项目构建流程

n8n 的 Docker 镜像构建分为两步：先编译应用，再打包镜像。

### 第一步：编译 n8n 应用

在项目根目录执行：

```bash
# 安装依赖
pnpm install --frozen-lockfile

# 构建所有包
pnpm build

# 创建生产部署产物（输出到 ./compiled 和 ./dist/task-runner-javascript）
pnpm run build:deploy
```

`build:deploy` 脚本（`scripts/build-n8n.mjs`）会：
1. 执行 `pnpm install` 和 `pnpm build`
2. 清理 package.json 中的前端/开发补丁
3. 使用 `pnpm deploy` 创建精简的生产部署到 `./compiled` 目录
4. 创建 JavaScript Task Runner 部署到 `./dist/task-runner-javascript`

### 第二步：构建 Docker 镜像

**一键构建（推荐）：**

```bash
pnpm build:docker
```

此命令会自动执行编译 + 镜像构建，生成 `n8nio/n8n:local` 和 `n8nio/runners:local` 两个镜像。

### 第三步：推送镜像到 ghcr.io

```bash
# 登录 ghcr.io（替换 YOUR_GITHUB_USERNAME 和 YOUR_GITHUB_PAT）
echo YOUR_GITHUB_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# 打 tag（替换 wanwanlu 为你的 GitHub 用户名）
docker tag n8nio/n8n:local ghcr.io/gitfot/my-n8n:latest

# 推送
docker push ghcr.io/gitfot/my-n8n:latest
```

> 首次推送后，镜像默认为**私有**。如果 HF Space 需要免认证拉取，需要在 GitHub 上将包设为公开：
> GitHub → Your Profile → Packages → `my-n8n` → Package settings → Change visibility → Public

<details>
<summary>如果使用 Docker Hub（备选）</summary>

```bash
docker login
docker tag n8nio/n8n:local wanwanlu/my-n8n:latest
docker push wanwanlu/my-n8n:latest
```

后续步骤中将 `ghcr.io/wanwanlu/my-n8n:latest` 替换为 `wanwanlu/my-n8n:latest` 即可。

</details>

---

## Docker Compose 部署

### 方案一：PostgreSQL（生产推荐）

适合生产环境，使用 PostgreSQL 作为数据库。

创建 `docker-compose.yml`：

```yaml
volumes:
  db_storage:
  n8n_data:

services:
  postgres:
    image: postgres:16
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER:-n8n}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB:-n8n}
    volumes:
      - db_storage:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER:-n8n} -d ${POSTGRES_DB:-n8n}']
      interval: 5s
      timeout: 5s
      retries: 10

  n8n:
    image: n8nio/n8n:local
    restart: always
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB:-n8n}
      - DB_POSTGRESDB_USER=${POSTGRES_USER:-n8n}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - WEBHOOK_URL=${WEBHOOK_URL:-http://localhost:5678/}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE:-Asia/Shanghai}
      - TZ=${TZ:-Asia/Shanghai}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
```

创建 `.env` 文件：

```env
# PostgreSQL
POSTGRES_USER=n8n
POSTGRES_PASSWORD=your-strong-postgres-password
POSTGRES_DB=n8n

# n8n
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/
GENERIC_TIMEZONE=Asia/Shanghai
TZ=Asia/Shanghai
N8N_ENCRYPTION_KEY=your-random-encryption-key-here
```

## 环境变量说明

### 核心配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `N8N_HOST` | n8n 实例的主机名 | `localhost` |
| `N8N_PORT` | 监听端口 | `5678` |
| `N8N_PROTOCOL` | 协议（http/https） | `http` |
| `WEBHOOK_URL` | Webhook 回调 URL | 自动生成 |
| `N8N_ENCRYPTION_KEY` | 凭据加密密钥（**必须固定**） | 自动生成 |
| `GENERIC_TIMEZONE` | n8n 时区（影响定时任务） | `America/New_York` |
| `TZ` | 系统时区 | 系统默认 |

### 数据库配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `DB_TYPE` | 数据库类型 | `sqlite`（可选 `postgresdb`） |
| `DB_POSTGRESDB_HOST` | PostgreSQL 主机 | `localhost` |
| `DB_POSTGRESDB_PORT` | PostgreSQL 端口 | `5432` |
| `DB_POSTGRESDB_DATABASE` | 数据库名 | `n8n` |
| `DB_POSTGRESDB_USER` | 数据库用户 | `postgres` |
| `DB_POSTGRESDB_PASSWORD` | 数据库密码 | - |
| `DB_POSTGRESDB_SCHEMA` | 数据库 Schema | `public` |

### Task Runner 配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `N8N_RUNNERS_MODE` | Runner 模式（`internal`/`external`） | `internal` |
| `N8N_RUNNERS_BROKER_LISTEN_ADDRESS` | Broker 监听地址 | `127.0.0.1` |
| `N8N_RUNNERS_AUTH_TOKEN` | Runner 认证令牌 | - |
| `N8N_RUNNERS_TASK_BROKER_URI` | Broker URI（Runner 侧配置） | `http://localhost:5679` |

### 其他常用配置

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `N8N_DIAGNOSTICS_ENABLED` | 是否发送匿名使用数据 | `true` |
| `N8N_METRICS` | 是否启用 Prometheus 指标 | `false` |
| `N8N_LOG_LEVEL` | 日志级别 | `info` |
| `N8N_USER_FOLDER` | 用户数据目录 | `/home/node/.n8n` |
| `EXECUTIONS_DATA_PRUNE` | 是否自动清理执行记录 | `true` |
| `EXECUTIONS_DATA_MAX_AGE` | 执行记录保留时间（小时） | `336`（14天） |

> 完整环境变量列表参见：https://docs.n8n.io/hosting/configuration/environment-variables/

### 使用文件传递敏感信息

为避免在环境变量中暴露敏感信息，以下变量支持 `_FILE` 后缀，从文件读取值（适用于 Docker Secrets）：

- `DB_POSTGRESDB_DATABASE_FILE`
- `DB_POSTGRESDB_HOST_FILE`
- `DB_POSTGRESDB_PASSWORD_FILE`
- `DB_POSTGRESDB_PORT_FILE`
- `DB_POSTGRESDB_USER_FILE`
- `DB_POSTGRESDB_SCHEMA_FILE`

示例：

```yaml
services:
  n8n:
    secrets:
      - db_password
    environment:
      - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## 数据持久化

### 关键数据目录

| 路径 | 说明 |
|------|------|
| `/home/node/.n8n` | n8n 用户数据（加密密钥、SQLite 数据库等） |
| `/var/lib/postgresql/data` | PostgreSQL 数据 |

> **重要**：即使使用 PostgreSQL，也必须持久化 `/home/node/.n8n` 目录。该目录包含加密密钥，丢失后已保存的凭据将无法解密。

### 自定义证书

如需信任自签名证书，将证书文件挂载到 `/opt/custom-certificates` 目录：

```yaml
services:
  n8n:
    volumes:
      - n8n_data:/home/node/.n8n
      - ./custom-certs:/opt/custom-certificates
```

n8n 的 entrypoint 脚本会自动检测并信任该目录下的证书。

---

## 反向代理配置

生产环境建议使用反向代理提供 HTTPS。

### Nginx 示例

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/ssl/certs/your-cert.pem;
    ssl_certificate_key /etc/ssl/private/your-key.pem;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

使用反向代理时，确保 `.env` 中配置：

```env
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/
```

---

## 维护与更新

### 更新流程

```bash
# 1. 拉取最新源码
git pull origin master

# 2. 重新构建镜像
pnpm build:docker

# 3. 将新镜像传输到服务器（如果在本地构建）

# 4. 在服务器上更新
docker compose down
docker compose up -d
```

### 备份

```bash
# 备份 PostgreSQL 数据
docker compose exec postgres pg_dump -U n8n n8n > backup_$(date +%Y%m%d).sql

# 备份 n8n 用户数据
docker compose cp n8n:/home/node/.n8n ./n8n-backup
```

### 查看日志

```bash
# 查看所有服务日志
docker compose logs -f

# 仅查看 n8n 日志
docker compose logs -f n8n
```

---

## 常见问题

### 1. 凭据解密失败

原因：`N8N_ENCRYPTION_KEY` 发生变化或 `/home/node/.n8n` 目录数据丢失。

解决：确保 `N8N_ENCRYPTION_KEY` 在 `.env` 中固定设置，并持久化 `/home/node/.n8n` 卷。

### 2. Webhook 不工作

确保 `WEBHOOK_URL` 设置为外部可访问的 URL，且反向代理正确转发 WebSocket 连接。

### 3. 时区不正确

同时设置 `GENERIC_TIMEZONE` 和 `TZ` 环境变量：

```env
GENERIC_TIMEZONE=Asia/Shanghai
TZ=Asia/Shanghai
```

### 4. 构建基础镜像问题

项目的 `docker/images/n8n-base/Dockerfile` 使用了内部基础镜像 `dhi.io/node:*`。如果你无法访问该镜像仓库，需要修改 `docker/images/n8n-base/Dockerfile`，将基础镜像替换为公共镜像，例如 `node:22-alpine`，并确保安装了必要的系统依赖（`tini`、`graphicsmagick`、`git`、`openssl` 等）。

不过，使用 `pnpm build:docker` 构建时，n8n 主镜像的 Dockerfile（`docker/images/n8n/Dockerfile`）引用的是 `n8nio/base` 镜像。如果你没有该基础镜像，可以先构建它：

```bash
docker build -t n8nio/base:22.22.0 -f docker/images/n8n-base/Dockerfile .
```

或者直接修改 `docker/images/n8n/Dockerfile` 中的 `FROM` 行，使用 `node:22-alpine` 并补充必要依赖。

### 5. 内存不足

n8n 在处理大量工作流时可能需要较多内存。可以通过 Docker Compose 限制和保障内存：

```yaml
services:
  n8n:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 512M
```
