# StreamHarbor Deploy

> **标准 Docker 部署：StreamHarbor + PostgreSQL + Redis + Celery Worker。**
> 此仓库只包含 NAS 部署所需的 Compose、配置模板与中文教程；不包含 StreamHarbor Python 源码、Cookie、授权文件或数据库。

## 部署前准备

- 一台已安装 **Docker** 和 **Docker Compose V2** 的 NAS / Linux 主机。
- 一个可访问 Docker Hub 的网络环境。
- 一个用于保存配置的宿主机目录，例如 `/vol1/1000/docker/streamharbor/config`。
- 一个 Emby 可读取、且容器也可读取的影视库目录，例如 `/vol1/1000/影视库`。

> 当前模板版本：`v1.0.0.10`。首次部署和每次升级都请使用固定版本号，不建议生产环境直接使用 `latest`。

## 三步启动

### 1. 准备文件

在 NAS 中创建部署目录，将本仓库的 `docker-compose.yml` 和 `.env.example` 放入该目录：

```bash
mkdir -p /opt/streamharbor
cd /opt/streamharbor
# 将下载的两个文件放在此目录后继续
cp .env.example .env
```

### 2. 编辑 `.env`

至少修改以下项目：

```dotenv
PUBLIC_BASE_URL=http://你的NAS地址:8000
CONFIG_PATH=/vol1/1000/docker/streamharbor/config
LIBRARY_ROOT=/vol1/1000/影视库
POSTGRES_PASSWORD=请替换为强密码
REDIS_PASSWORD=请替换为强密码
GATEWAY_SECRET=请替换为至少32位的随机字符串
```

`你的NAS地址` 例如 `192.168.50.220`。不要把真实 Cookie、密码、授权文件或 `.env` 上传到 GitHub。

### 3. 启动服务

```bash
cd /opt/streamharbor
docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

浏览器打开：`http://你的NAS地址:8000`。

## 容器说明

| 容器 | 作用 |
| --- | --- |
| `streamharbor` | Web 管理页面、115 网关与 Emby 反向代理。 |
| `streamharbor-worker` | Celery 后台任务 Worker，负责耗时的导入、校验和 STRM 生成任务。 |
| `streamharbor-postgres` | PostgreSQL 数据库，保存资源、分享、任务和配置数据。 |
| `streamharbor-redis` | Redis 缓存，以及 Celery 的消息队列和结果后端。 |

`streamharbor-worker` 不是重复的 Web 容器。它只处理后台队列任务；保留它可以避免导入或生成 STRM 时阻塞管理页面。

## 常用命令

```bash
# 查看状态
docker compose ps

# 查看应用日志
docker compose logs --tail=200 app

# 查看后台任务日志
docker compose logs --tail=200 worker

# 重启应用和 Worker
docker compose restart app worker

# 停止服务，但保留 PostgreSQL、Redis 与配置数据
docker compose down
```

**不要执行 `docker compose down -v`**，否则会删除 PostgreSQL 和 Redis 的命名卷，导致数据丢失。

## 更新版本

编辑 `.env` 的版本号后执行：

```dotenv
STREAMHARBOR_VERSION=1.0.0.10
```

```bash
docker compose pull
docker compose up -d
docker compose ps
```

## 详细教程

- [Docker 部署教程（含结构图与排错）](docs/DOCKER_DEPLOY_ZH.md)
- [Docker Hub 镜像仓库](https://hub.docker.com/r/noway2004/streamharbor)
- [GitHub Releases](https://github.com/noway2004/StreamHarbor/releases)

## 安全说明

- `.env`、Cookie、授权文件和数据库均为私密数据，不要提交到 GitHub。
- PostgreSQL 与 Redis 端口默认只绑定在 NAS 本机 `127.0.0.1`，不会暴露到局域网。
- `GATEWAY_SECRET` 用于签名播放和探测链接；部署后请保持稳定，不要随意更换。更换后需要重新生成 STRM 并扫描 Emby 媒体库。
