# StreamHarbor Docker 简单部署教程

Docker Hub 镜像：`noway2004/streamharbor:v1.0.0.6`

本教程按照常见 NAS Compose 模板编写：**一个服务、列表式环境变量、宿主机目录映射、bridge 网络和端口映射**。StreamHarbor 默认使用本机 SQLite 数据库和进程内缓存，不需要另外安装 PostgreSQL、Redis 或 Celery。

![StreamHarbor 单容器部署结构](images/docker-architecture.svg)

## 1. 部署结构

```text
浏览器 ── 8000 ──> StreamHarbor
Emby   ── 8097 ──> StreamHarbor Emby 反向代理
                       │
                       ├── /app/runtime  -> NAS 的 config 目录
                       └── 媒体库目录     -> NAS 的影视库目录
```

- Web 管理端口：`8000`
- Emby 反向代理端口：`8097`
- 数据库：`/app/runtime/pan115.db`（SQLite）
- 缓存/任务：应用本机内存，`TASK_MODE=inline`
- 配置保存目录：`/vol1/1000/docker/streamharbor/config`
- STRM 媒体库：`/vol1/1000/影视库`

![部署文件与数据位置](images/docker-files.svg)

## 2. 创建目录

在 NAS SSH 终端执行：

```bash
mkdir -p /vol1/1000/docker/streamharbor/config
mkdir -p /vol1/1000/影视库
mkdir -p /opt/streamharbor
cd /opt/streamharbor
```

如果你的 NAS 目录不同，后面修改 `.env` 中的 `CONFIG_PATH` 和 `LIBRARY_ROOT`。

## 3. 准备两个文件

把 GitHub Release 中的以下文件放到 `/opt/streamharbor/`：

```text
docker-compose.yml
.env.example
```

复制配置模板：

```bash
cp .env.example .env
```

编辑 `.env`：

```env
STREAMHARBOR_VERSION=1.0.0.6
TZ=Asia/Shanghai
APP_PORT=8000
EMBY_PROXY_PORT=8097
PUBLIC_BASE_URL=http://你的NAS地址:8000
CONFIG_PATH=/vol1/1000/docker/streamharbor/config
LIBRARY_ROOT=/vol1/1000/影视库
GATEWAY_SECRET=请改成至少32位的随机字符串
```

只需要重点修改三项：

1. `PUBLIC_BASE_URL`：把“你的NAS地址”替换为 NAS 的真实 IP 或域名。
2. `CONFIG_PATH`、`LIBRARY_ROOT`：路径和你的 NAS 实际目录保持一致。
3. `GATEWAY_SECRET`：换成至少 32 位随机字符串，不要把真实值上传到 GitHub。

## 4. Compose 模板

项目中的 `docker-compose.yml` 已改成和常见 NAS 应用相同的直观结构：

```yaml
version: "3"

services:
  app:
    image: noway2004/streamharbor:v${STREAMHARBOR_VERSION:-1.0.0.6}
    container_name: streamharbor
    volumes:
      - "${CONFIG_PATH:-/vol1/1000/docker/streamharbor/config}:/app/runtime"
      - "${LIBRARY_ROOT:-/vol1/1000/影视库}:${LIBRARY_ROOT:-/vol1/1000/影视库}"
    environment:
      - TZ=${TZ:-Asia/Shanghai}
      - APP_ENV=production
      - APP_VERSION=${STREAMHARBOR_VERSION:-1.0.0.6}
      - PUBLIC_BASE_URL=${PUBLIC_BASE_URL:-http://127.0.0.1:8000}
      - DATABASE_URL=sqlite:////app/runtime/pan115.db
      - TASK_MODE=inline
      - PROVIDER_MODE=cookie
      - GATEWAY_SECRET=${GATEWAY_SECRET:-change-this-secret-before-public-access}
      - LIBRARY_ROOT=${LIBRARY_ROOT:-/vol1/1000/影视库}
      - LIBRARY_ALLOWED_ROOT=${LIBRARY_ROOT:-/vol1/1000/影视库}
      - ENABLE_DOCS=false
    ports:
      - "${APP_PORT:-8000}:8000"
      - "${EMBY_PROXY_PORT:-8097}:8097"
    network_mode: bridge
    restart: always
```

这里没有 PostgreSQL、Redis 的地址、账号或密码。数据库、Cookie、Web 配置和授权状态都保存在宿主机 `CONFIG_PATH` 目录中。

## 5. 启动

```bash
cd /opt/streamharbor
docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

查看日志：

```bash
docker compose logs -f app
```

浏览器打开：

```text
http://你的NAS地址:8000
```

容器显示 `Up` 后，在 Web 页面中填写 115 Cookie、授权码、Telegram 和媒体库配置。真实 Cookie、授权信息和密码只在自己的 NAS 上填写。

## 6. Emby 反向代理设置

本模板使用 `bridge` 网络并映射 `8097:8097`，所以 Web 页面中的 Emby 反向代理端口请填写 `8097`。

重要说明：

- StreamHarbor 容器访问 NAS 上的 Emby 时，Emby 地址不要填写 `127.0.0.1:8096`。
- 应填写 NAS 的真实局域网地址，例如 `http://192.168.1.10:8096`。
- 如果要把反代端口改成 `9000`，同时把 `.env` 改为 `EMBY_PROXY_PORT=9000`，并在 Web 页面中也填写 `9000`。

## 7. 更新版本

更新 `.env` 中的版本号后执行：

```bash
cd /opt/streamharbor
docker compose pull
docker compose up -d
docker compose ps
```

例如：

```env
STREAMHARBOR_VERSION=1.0.0.6
```

不要执行：

```bash
docker compose down -v
```

本模板使用宿主机绑定目录，即使重建容器也会保留数据；但仍建议升级前备份：

```bash
cp -a /vol1/1000/docker/streamharbor/config \
  /vol1/1000/docker/streamharbor/config-backup-$(date +%Y%m%d-%H%M%S)
```

## 8. 常用命令

```bash
# 状态
docker compose ps

# 日志
docker compose logs --tail=200 app

# 重启
docker compose restart app

# 停止但保留数据
docker compose down

# 再次启动
docker compose up -d
```

## 9. 常见问题

### 页面打不开

检查 NAS 防火墙是否允许 `8000`，再执行：

```bash
docker compose ps
docker compose logs --tail=200 app
```

### SQLite 提示无法写入

确认 `CONFIG_PATH` 存在且 Docker 有写权限：

```bash
ls -ld /vol1/1000/docker/streamharbor/config
```

### 媒体库没有 STRM

确认 Compose 左右两侧媒体库路径相同，并且 Web 页面中的媒体库根目录也使用同一路径，例如 `/vol1/1000/影视库`。

### Emby 反代启动但外部访问失败

确认 `.env` 的 `EMBY_PROXY_PORT`、Compose 端口映射和 Web 页面填写的反代端口三者一致。

## 10. 镜像来源

- 固定版本：`noway2004/streamharbor:v1.0.0.6`
- 自动更新标签：`noway2004/streamharbor:latest`
- 正式部署建议使用固定版本，升级时手动修改版本号，便于回退。
- 公开镜像是 Cython 加密发布镜像，不要上传未加密开发镜像。
