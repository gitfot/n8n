# 部署自定义 n8n 到 Hugging Face Spaces

本文档描述如何将修改过源码的 n8n 部署到 Hugging Face Spaces（Docker SDK）。

提供两种部署方案：

| 方案 | 思路                                                            | 适用场景                             |
| ------ | ----------------------------------------------------------------- | -------------------------------------- |
| [方案 A：本地预编译](#方案-a本地预编译)     | 本地编译 → 编译产物推送到 HF git 仓库 → HF 构建轻量运行时镜像 | 不想维护 Docker Hub 账号，改动不频繁 |
| [方案 C：远程镜像仓库中转](#方案-c远程镜像仓库中转)     | 本地编译 → 构建 Docker 镜像 → 推送到 ghcr.io → HF 直接拉取   | 最简单快速，推荐使用 ghcr.io         |

## 通用前置条件

* Hugging Face 账号，已创建 Docker 类型的 Space（例如 `wanwanlu/my-n8n`）
* HF Access Token（[生成地址](https://huggingface.co/settings/tokens)），需要 write 权限
* Supabase 免费 PostgreSQL 数据库（[注册](https://supabase.com/dashboard/sign-up)），用于数据持久化
* 本地 n8n 源码可正常编译（Node.js >= 22.16，pnpm >= 10.22.0）

## 通用步骤：准备 Supabase 数据库

HF Spaces 免费层的磁盘是临时性的，Space 重启后本地文件全部丢失。必须使用外部数据库。

1. 登录 [Supabase Dashboard](https://supabase.com/dashboard)，创建新项目，**记住数据库密码**
2. 项目创建完成后，点击顶部导航栏的 **Connect** 按钮
3. 选择 **Transaction pooler** 模式，记录以下信息：

	* `Host`（例如 `aws-0-ap-southeast-1.pooler.supabase.com`）
	* `Port`（通常是 `6543`）
	* `User`（例如 `postgres.xxxxxxxxxxxx`）
	* `Database`（通常是 `postgres`）

---


## 方案1：远程镜像仓库中转 (推荐)

本地编译并构建 Docker 镜像，推送到远程镜像仓库，HF Space 的 Dockerfile 直接 `FROM` 拉取。

推荐使用 **ghcr.io**（GitHub Container Registry）而非 Docker Hub：

| 对比项         | Docker Hub                           | ghcr.io            |
| ---------------- | -------------------------------------- | -------------------- |
| 免费私有镜像   | 1 个                                 | 无限制             |
| 拉取速率限制   | 100 次/6h（匿名）、200 次/6h（登录） | 无限制             |
| 与 GitHub 集成 | 无                                   | 原生绑定，权限复用 |
| 认证方式       | Docker Hub 账号                      | GitHub PAT         |

> HF Spaces 每次构建/重启都会拉取镜像，Docker Hub 的速率限制容易成为瓶颈。

**额外前置条件**：

* 本地已安装 Docker（或 Podman）
* GitHub 账号 + Personal Access Token（需要 `write:packages` 权限）

	* 生成地址：[GitHub Settings → Tokens](https://github.com/settings/tokens)
	* 勾选 `write:packages` 和 `read:packages` 权限

### 1.1. 本地编译并构建 Docker 镜像

```bash
# 编译 n8n 并构建 Docker 镜像（一条命令完成）
pnpm build:docker
```

> 该命令会先执行 `build:deploy` 生成 `compiled/`，再执行 `dockerize-n8n.mjs` 构建镜像。
> 构建完成后本地会生成 `n8nio/n8n:local` 镜像。

验证镜像：

```bash
docker images n8nio/n8n:local
```

### 1.2. 推送镜像到 ghcr.io

```bash
# 登录 ghcr.io（替换 YOUR_GITHUB_USERNAME 和 YOUR_GITHUB_PAT）
echo YOUR_GITHUB_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# 打 tag（替换 wanwanlu 为你的 GitHub 用户名）
docker tag n8nio/n8n:local ghcr.io/wanwanlu/my-n8n:latest

# 推送
docker push ghcr.io/wanwanlu/my-n8n:latest
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

### 1.3. 准备 HF Space 仓库

```bash
git clone https://huggingface.co/spaces/wanwanlu/my-n8n
cd my-n8n
```

#### 创建 `README.md`

```yaml
---
title: my-n8n
emoji: 🤖
colorFrom: pink
colorTo: indigo
sdk: docker
pinned: false
app_port: 5678
short_description: n8n workflow automation (custom build)
---
```

#### 创建 `Dockerfile`

```dockerfile
FROM ghcr.io/wanwanlu/my-n8n:latest

# HF Spaces 以 UID 1000 运行，n8n 镜像的 node 用户正好是 1000
USER 1000

EXPOSE 5678

ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]
```

> 只需这几行。所有运行时依赖、编译产物都已包含在你推送的镜像中。

#### 最终目录结构

```
my-n8n/
├── README.md
└── Dockerfile
```

### 1.4. 推送部署

```bash
git add README.md Dockerfile
git commit -m "Add custom n8n build via ghcr.io"
git push
```

HF 会拉取你的 ghcr.io 镜像并启动，通常 1-3 分钟完成。

### 1.5. 配置环境变量

与方案 A 完全相同，在 HF Space 的 **Settings → Variables and secrets** 中添加：

#### Variables（明文变量）

| 变量名 | 值 | 说明                    |
| -------- | ---- | ------------------------- |
| `DB_TYPE`       | `postgresdb`   | 使用 PostgreSQL         |
| `DB_POSTGRESDB_HOST`       | `你的 Supabase Host`   | 数据库主机地址          |
| `DB_POSTGRESDB_PORT`       | `6543`   | Transaction pooler 端口 |
| `DB_POSTGRESDB_DATABASE`       | `postgres`   | 数据库名                |
| `DB_POSTGRESDB_USER`       | `postgres.xxxx`   | Supabase 用户名         |
| `WEBHOOK_URL`       | `https://wanwanlu-my-n8n.hf.space/`   | Webhook 回调地址        |
| `N8N_EDITOR_BASE_URL`       | `https://wanwanlu-my-n8n.hf.space/`   | 编辑器基础 URL          |
| `GENERIC_TIMEZONE`       | `Asia/Shanghai`   | n8n 时区                |
| `TZ`       | `Asia/Shanghai`   | 系统时区                |

#### Secrets（加密变量）

| 变量名 | 值                  | 说明                 |
| -------- | --------------------- | ---------------------- |
| `DB_POSTGRESDB_PASSWORD`       | Supabase 数据库密码 |                      |
| `N8N_ENCRYPTION_KEY`       | 随机字符串          | **必填**，用于加密存储的凭据 |

### 1.6. 访问 n8n

部署成功后访问：

```
https://wanwanlu-my-n8n.hf.space/
```

### 1.7. 更新部署

当源码有改动时：

```bash
# 1. 重新编译并构建镜像
pnpm build:docker

# 2. 打 tag 并推送到 ghcr.io
docker tag n8nio/n8n:local ghcr.io/wanwanlu/my-n8n:latest
docker push ghcr.io/wanwanlu/my-n8n:latest

# 3. 在 HF Space 页面点击 "Factory reboot" 触发重新拉取镜像
#    或者在 HF Space 仓库中做一次空提交触发重建：
cd my-n8n
git commit --allow-empty -m "Trigger rebuild"
git push
```

---

## 方案2：本地预编译

本地编译 n8n，将编译产物 `compiled/` 推送到 HF Space 的 git 仓库，由 HF 构建仅包含运行时的轻量镜像。

**额外前置条件**：本地已安装 Git LFS（`compiled/` 体积较大，需要 LFS 支持）。

### 2.1. 本地编译 n8n

在项目根目录执行：

```bash
# 编译 n8n，生成 compiled/ 目录
pnpm build:deploy
```

> 编译时间取决于机器性能，通常 5-15 分钟。完成后项目根目录下会出现 `compiled/` 目录（约 200-400MB）。

验证编译产物：

```bash
ls -la compiled/
# 应包含 bin/、node_modules/、package.json 等
ls compiled/bin/n8n
# 应存在此入口文件
```

### 2.2. 准备 HF Space 仓库

#### 克隆 Space

```bash
git clone https://huggingface.co/spaces/wanwanlu/my-n8n
cd my-n8n
```

克隆时会提示输入密码，使用 HF Access Token 作为密码。

#### 初始化 Git LFS

`compiled/` 目录包含大量文件，必须使用 Git LFS：

```bash
git lfs install
```

#### 创建 `.gitattributes`

```
compiled/** filter=lfs diff=lfs merge=lfs -text
```

#### 创建 `README.md`

```yaml
---
title: my-n8n
emoji: 🤖
colorFrom: pink
colorTo: indigo
sdk: docker
pinned: false
app_port: 5678
short_description: n8n workflow automation (custom build)
---
```

> `app_port: 5678` 告诉 HF 将外部流量转发到容器的 5678 端口（n8n 默认端口）。

#### 创建 `Dockerfile`

```dockerfile
FROM node:22-alpine

# 安装运行时依赖
RUN apk add --no-cache \
    git \
    openssh \
    openssl \
    tini \
    tzdata \
    ca-certificates \
    libc6-compat \
    && npm install -g full-icu@1.5.0 \
    && rm -rf /tmp/* /root/.npm

ENV NODE_ENV=production \
    NODE_ICU_DATA=/usr/local/lib/node_modules/full-icu \
    SHELL=/bin/sh

WORKDIR /home/node

# 复制预编译好的 n8n
COPY --chown=1000:1000 ./compiled /usr/local/lib/node_modules/n8n
COPY --chown=1000:1000 ./docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

# 重建 sqlite3 原生模块、创建 n8n 命令符号链接、设置数据目录权限
RUN cd /usr/local/lib/node_modules/n8n && \
    npm rebuild sqlite3 && \
    ln -s /usr/local/lib/node_modules/n8n/bin/n8n /usr/local/bin/n8n && \
    mkdir -p /home/node/.n8n && \
    chown -R 1000:1000 /home/node && \
    rm -rf /root/.npm /tmp/*

# n8n 默认端口
EXPOSE 5678

# HF Spaces 以 UID 1000 运行容器，与 node 用户一致
USER 1000

ENTRYPOINT ["tini", "--", "/docker-entrypoint.sh"]
```

#### 创建 `docker-entrypoint.sh`

```bash
#!/bin/sh
if [ -d /opt/custom-certificates ]; then
  echo "Trusting custom certificates from /opt/custom-certificates."
  export NODE_OPTIONS="--use-openssl-ca $NODE_OPTIONS"
  export SSL_CERT_DIR=/opt/custom-certificates
  c_rehash /opt/custom-certificates
fi

if [ "$#" -gt 0 ]; then
  exec n8n "$@"
else
  exec n8n
fi
```

#### 复制编译产物

```bash
# 从 n8n 项目根目录复制编译产物（根据实际路径调整）
cp -r /path/to/n8n/compiled ./compiled
```

#### 最终目录结构

```
my-n8n/
├── .gitattributes
├── README.md
├── Dockerfile
├── docker-entrypoint.sh
└── compiled/
    ├── bin/
    ├── node_modules/
    ├── package.json
    └── ...
```

### 2.3. 推送部署

```bash
git add .gitattributes README.md Dockerfile docker-entrypoint.sh compiled/
git commit -m "Add custom n8n build"
git push
```

推送完成后，HF 会自动触发 Docker 构建。可以在 Space 页面的 **Logs** 标签查看构建进度，通常 3-5 分钟完成。

### 2.4. 配置环境变量

在 HF Space 页面进入 **Settings → Variables and secrets**，添加以下配置：

### Variables（明文变量）

| 变量名 | 值 | 说明                    |
| -------- | ---- | ------------------------- |
| `DB_TYPE`       | `postgresdb`   | 使用 PostgreSQL         |
| `DB_POSTGRESDB_HOST`       | `你的 Supabase Host`   | 数据库主机地址          |
| `DB_POSTGRESDB_PORT`       | `6543`   | Transaction pooler 端口 |
| `DB_POSTGRESDB_DATABASE`       | `postgres`   | 数据库名                |
| `DB_POSTGRESDB_USER`       | `postgres.xxxx`   | Supabase 用户名         |
| `WEBHOOK_URL`       | `https://wanwanlu-my-n8n.hf.space/`   | Webhook 回调地址        |
| `N8N_EDITOR_BASE_URL`       | `https://wanwanlu-my-n8n.hf.space/`   | 编辑器基础 URL          |
| `GENERIC_TIMEZONE`       | `Asia/Shanghai`   | n8n 时区                |
| `TZ`       | `Asia/Shanghai`   | 系统时区                |

### Secrets（加密变量）

| 变量名 | 值                  | 说明                 |
| -------- | --------------------- | ---------------------- |
| `DB_POSTGRESDB_PASSWORD`       | Supabase 数据库密码 |                      |
| `N8N_ENCRYPTION_KEY`       | 随机字符串          | **必填**，用于加密存储的凭据 |

生成 `N8N_ENCRYPTION_KEY`：

```bash
openssl rand -base64 32
```

> **重要**：`N8N_ENCRYPTION_KEY` 一旦设定后不要更改，否则已保存的所有凭据将无法解密。

### 2.5. 访问 n8n

部署成功后，直接在浏览器中访问：

```
https://wanwanlu-my-n8n.hf.space/
```

首次访问会进入 n8n 初始化页面，创建管理员账号即可开始使用。

> **注意**：不要在 HF Space 页面内嵌的 iframe 中使用 n8n。n8n 使用 helmet 设置了 `X-Frame-Options: sameorigin`，iframe 中会被阻止。请直接访问上述 URL。

### 2.6. 更新部署

当源码有改动时，重复以下步骤：

```bash
# 1. 重新编译
pnpm build:deploy

# 2. 进入 HF Space 仓库目录
cd my-n8n

# 3. 删除旧产物，复制新产物（根据实际路径调整）
rm -rf compiled
cp -r /path/to/n8n/compiled ./compiled

# 4. 提交推送
git add compiled/
git commit -m "Update n8n build"
git push
```

---

## 注意事项

### 数据持久化

* HF Spaces 免费层磁盘是**临时性**的，容器重启后本地文件全部丢失
* 必须使用外部 PostgreSQL（如 Supabase），否则工作流、凭据等数据会丢失
* `N8N_ENCRYPTION_KEY` 必须固定不变，否则重启后已有凭据无法解密

### 休眠机制

* 免费 Space 在 **48 小时无流量**后会自动休眠（Sleep），容器停止运行
* 收到新请求时会自动唤醒，但冷启动需要 1-2 分钟
* Cron/Schedule 类工作流在休眠期间**不会执行**
* 如需保持活跃，可使用 [UptimeRobot](https://uptimerobot.com/) 等服务每 30 分钟 ping 一次

### 资源限制

| 资源 | 免费层额度    |
| ------ | --------------- |
| CPU  | 2 vCPU        |
| 内存 | 16 GB         |
| 磁盘 | 50 GB（临时） |

对个人使用和中小规模工作流完全够用。

### Git LFS 存储

* HF 免费账户的 LFS 存储有限额，`compiled/` 每次更新都会占用额外空间
* 如果遇到 LFS 存储不足，可考虑切换为 [方案 C（远程镜像仓库中转）](#方案-c远程镜像仓库中转) 或付费升级 HF 存储

### WEBHOOK_URL 格式

* URL 格式为 `https://{用户名}-{space名}.hf.space/`
* 用户名中的 `_` 会被替换为 `-`
* 末尾的 `/` 不可省略
