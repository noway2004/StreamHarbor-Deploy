# StreamHarbor Deploy

**仅包含部署文件，不包含 StreamHarbor 的 Python 源码、授权服务源码、测试、构建脚本、数据库、Cookie、私钥或任何真实凭据。**

此仓库用于从 Docker Hub 部署 StreamHarbor 的 Cython 加密镜像。

- Docker Hub：`noway2004/streamharbor`
- 当前稳定版本：`v1.0.0.6`
- 私有源码仓库：仅维护者可访问，不在此公开仓库中。

![部署文件与数据保存位置](docs/images/docker-files.svg)

## 快速部署

```bash
mkdir -p /vol1/1000/docker/streamharbor/config
mkdir -p /vol1/1000/影视库
mkdir -p /opt/streamharbor
cd /opt/streamharbor

# 下载本仓库中的 docker-compose.yml 和 .env.example 后：
cp .env.example .env
# 编辑 .env：填写 NAS 地址、实际目录和随机 GATEWAY_SECRET

docker compose pull
docker compose up -d
docker compose ps
```

访问：`http://你的NAS地址:8000`

完整图文教程请阅读：[Docker 简单部署教程](docs/DOCKER_DEPLOY_ZH.md)。

## 仓库允许的文件

- `docker-compose.yml`
- `.env.example`（仅示例值）
- 部署说明和图片
- 加密离线镜像包及其 `SHA256SUMS`（仅在 GitHub Release Assets 中提供）

## 绝不应提交的文件

```text
app/
license_server/
tests/
scripts/
Dockerfile*
*.py
.env
runtime/
release/
*.pem
*.key
数据库、Cookie、授权状态、私钥、真实密码或 Token
```

## 更新

将 `.env` 的 `STREAMHARBOR_VERSION` 改为要使用的固定版本，例如：

```env
STREAMHARBOR_VERSION=1.0.0.6
```

然后运行：

```bash
docker compose pull
docker compose up -d
```

不要执行 `docker compose down -v`。升级前请备份 `CONFIG_PATH` 对应的宿主机目录。