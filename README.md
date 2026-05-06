# TenJudge 单机部署说明

本仓库提供 TenJudge 的 Docker Compose 单机部署方案，包含前端、后端、判题服务、沙箱和基础中间件。

## 服务组成

Compose 会启动以下服务：

- `tenjudge-frontend`：前端静态站点，容器内 Nginx 托管页面，并把 `/api/` 代理到后端。
- `tenjudge-server`：后端 API 服务。
- `tenjudge-judge`：判题 worker，消费 RabbitMQ 中的评测任务。
- `tenjudge-sandbox`：go-judge 沙箱服务。
- `postgres`：业务数据库，首次初始化时执行 `db/` 下的 SQL。
- `redis`：缓存、登录态、分布式锁。
- `rabbitmq`：提交评测任务队列。
- `minio`：题目数据、提交代码等对象存储。

## 前置条件

部署前需要先准备好 Docker 和 Docker Compose。默认会使用本地 TenJudge 镜像：

```bash
tenjudge-server:latest
tenjudge-judge:latest
tenjudge-sandbox:latest
tenjudge-frontend:latest
```

部署过程不会执行 Dockerfile，也不会自动构建镜像。

## 配置 `.env`

启动前先编辑 `.env`，至少替换所有密码占位值。

TenJudge 自有镜像默认使用本地镜像；如需切换镜像仓库，配置镜像前缀即可，前缀需要以 `/` 结尾：

```env
TENJUDGE_IMAGE_PREFIX=
TENJUDGE_IMAGE_PREFIX=ghcr.io/tenjudge/
TENJUDGE_IMAGE_PREFIX=ccr.ccs.tencentyun.com/tenjudge/
```

密码配置：

```env
POSTGRES_PASSWORD=your_real_postgres_password
REDIS_PASSWORD=your_real_redis_password
RABBITMQ_PASSWORD=your_real_rabbitmq_password
MINIO_ROOT_PASSWORD=your_real_minio_password
```

后端和判题服务时区配置：

```env
TZ=Asia/Shanghai
```

超级管理员初始化配置：

```env
APP_SUPER_ADMIN_USERNAME=admin
APP_SUPER_ADMIN_PASSWORD=change_me_super_admin_password
APP_SUPER_ADMIN_EMAIL=admin@example.com
```

判题并发配置：

```env
APP_RABBITMQ_SUBMIT_LISTENER_CONCURRENCY=2-3
```

前端默认只监听宿主机本机地址：

```env
FRONTEND_SERVER_NAME=localhost
FRONTEND_HTTP_BIND=127.0.0.1
FRONTEND_HTTP_PORT=3000
```

默认访问地址为：

```text
http://127.0.0.1:3000
```

## 启动

启动服务：

```bash
docker compose up -d
```

查看服务状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f
```

只查看某个服务日志，例如后端：

```bash
docker compose logs -f tenjudge-server
```

## 配置校验

修改 compose 或 `.env` 后，可以先运行：

```bash
docker compose config
```

该命令只校验配置和变量插值，不会启动服务。

## 端口说明

默认端口绑定如下：

- 前端：`127.0.0.1:3000 -> tenjudge-frontend:80`
- 后端：`127.0.0.1:8080 -> tenjudge-server:8080`
- 沙箱：`127.0.0.1:5050 -> tenjudge-sandbox:5050`
- PostgreSQL：`127.0.0.1:5432 -> postgres:5432`
- Redis：`127.0.0.1:6379 -> redis:6379`
- RabbitMQ：`127.0.0.1:5672 -> rabbitmq:5672`
- RabbitMQ 管理后台：`127.0.0.1:15672 -> rabbitmq:15672`
- MinIO API：`127.0.0.1:9000 -> minio:9000`
- MinIO Console：`127.0.0.1:9001 -> minio:9001`

这些端口默认都只绑定 `127.0.0.1`，不会直接暴露到公网。

## 前端和 API 代理

前端镜像使用 Nginx 托管已构建好的 `dist`。`nginx/frontend.conf.template` 会被挂载到前端容器中，负责：

- Vue Router history 模式回退到 `index.html`。
- 将 `/api/` 代理到 `http://tenjudge-server:8080/`。
- 转发到后端时去掉 `/api` 前缀。

前端生产构建默认使用：

```env
VITE_API_BASE_URL=/api
```

因此浏览器访问前端页面后，请求会走同域名下的 `/api/...`，再由前端容器内的 Nginx 转发到后端。

## HTTPS 和公网访问

当前 compose 不直接配置 HTTPS，也不直接对公网开放前端端口。推荐生产结构：

```text
用户浏览器
-> https://你的域名
-> 宿主机 Nginx / Caddy / 宝塔 / 云负载均衡
-> http://127.0.0.1:3000
-> tenjudge-frontend
-> /api/ 代理到 tenjudge-server
```

如果要支持 HTTPS，需要在外层网关上完成：

- 域名 DNS 解析到服务器公网 IP。
- 开放服务器安全组或防火墙的 `80` 和 `443`。
- 签发并配置 TLS 证书。
- 将 HTTPS 请求反向代理到 `http://127.0.0.1:3000`。

前端代码本身支持 HTTPS，因为生产 API 地址是相对路径 `/api`，通过 HTTPS 域名访问时 API 请求也会自动使用 HTTPS。

## 数据初始化

PostgreSQL 首次创建数据卷时，会自动执行 `db/` 下的初始化 SQL。

如果数据库卷已经存在，PostgreSQL 不会重复执行初始化脚本。需要重新初始化时，应先确认数据可以删除，再移除对应 Docker volume。

## 停止和重启

停止服务：

```bash
docker compose down
```

停止服务并删除所有容器、网络和本项目创建的 volume：

```bash
docker compose down -v
```

该命令会删除 PostgreSQL、Redis、RabbitMQ、MinIO 的持久化数据。只有在确认不需要保留数据，或需要彻底重新初始化环境时再执行。

重启服务：

```bash
docker compose restart
```

更新某个已重新构建的镜像后，重建对应容器：

```bash
docker compose up -d --force-recreate tenjudge-server
```

## 注意事项

- `.env` 中的密码和超级管理员密码必须在生产部署前替换。
- `tenjudge-sandbox` 使用 `privileged: true`，应只在可信服务器上运行。
- 默认 compose 适合单机部署，不包含多机扩展、备份、监控和日志采集。
- 不要随意删除 Docker volume；PostgreSQL、Redis、RabbitMQ、MinIO 的持久化数据都存放在 volume 中。
- 服务启动后也不要删除本仓库。后续执行 `docker compose ps`、`logs`、`down`、`up`、`restart` 仍需要 `.env` 和 `docker-compose.yml`；前端容器重启或重建时还需要 `nginx/frontend.conf.template`。
- 数据主要存放在 Docker volume 中，不在本仓库中；但本仓库是部署配置和运维入口，应作为部署资产保留。
