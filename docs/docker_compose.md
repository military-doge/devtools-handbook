# Docker Compose

> 官方文档：<https://docs.docker.com/compose/>  
> 配置文件参考：<https://docs.docker.com/reference/compose-file/>  

单容器用 `docker run` 足够；一旦涉及多个容器、固定端口与挂载参数、反复启停，手写命令既长又容易漏参数。Compose 把整套服务写进一个 YAML 文件，用一条命令创建、更新、停止和删除，配置可版本化、可复用。本节使用 Vaultwarden 自托管密码管理服务器的部署进行演示。

## 一、安装 Compose

> 按 [docker_install.md](docker_install.md) 的步骤安装时，`docker-compose-plugin` 已包含在安装命令中，本步可直接跳过。

查看 compose 是否安装：

```bash
docker compose version  # 输出类似 Docker Compose version vX.X.X 为已安装
```

未安装则运行以下命令（Ubuntu/Debian）：

```bash
sudo apt update
sudo apt install docker-compose-plugin
```

## 二、本地目录结构

```text
/srv/<项目名>/
├── docker-compose.yaml   # 配置文件
├── .env                  # 环境变量（密码、令牌等，注意写入 .gitignore 防止上传到云端）
├── app1_data/              # 应用 1 数据
└── app2_data/              # 应用 2 数据
```

本文建议将项目放在 `/srv/<项目名>`，如果不提供对外服务可以选择存放在其他位置，数据挂载也可选择其他合适的位置。

## 三、部署 Vaultwarden（数据库选用 PostgreSQL）

Vaultwarden 是兼容 Bitwarden 客户端的轻量服务端。它默认可以使用内嵌的 SQLite；本示例为了展示 Compose 的多服务编排，改用 PostgreSQL。

附件、RSA 密钥和配置存放在宿主机的 `/srv/vaultwarden/vw_data`，PostgreSQL 数据存放在 `/srv/vaultwarden/db_data`，两者都通过绑定挂载持久化。要省事可以使用默认的 SQLite。

### 1. 准备目录

```bash
sudo mkdir /srv/vaultwarden
cd /srv/vaultwarden
```

这里只创建项目目录；`vw_data`、`db_data` 会在首次 `docker compose up -d` 时由 Docker 自动创建，分别存放 Vaultwarden 的附件/密钥/配置和 PostgreSQL 数据。

### 2. 编写 compose.yaml

先创建文件并用编辑器打开：

```bash
sudo touch docker-compose.yaml
sudo nano docker-compose.yaml
```

编写（黏贴）以下内容到编辑器中保存并退出：

```yaml
name: vaultwarden

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    ports:
      - "127.0.0.1:8080:80"
    environment:
      DOMAIN: "https://<主机名>.<tailnet>.ts.net"
      ADMIN_TOKEN: ${ADMIN_TOKEN:?请在 .env 中设置 ADMIN_TOKEN}
      SIGNUPS_ALLOWED: "true"
      DATABASE_URL: "postgresql://vaultwarden:${POSTGRES_PASSWORD:?请在 .env 中设置 POSTGRES_PASSWORD}@db:5432/vaultwarden"
    volumes:
      - /srv/vaultwarden/vw_data:/data
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost/alive"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  db:
    image: postgres:18
    container_name: vaultwarden-db
    environment:
      POSTGRES_DB: vaultwarden
      POSTGRES_USER: vaultwarden
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?请在 .env 中设置 POSTGRES_PASSWORD}
    volumes:
      - /srv/vaultwarden/db_data:/var/lib/postgresql
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U vaultwarden -d vaultwarden"]
      interval: 10s
      timeout: 5s
      retries: 5
```

配置说明：

- `ports`：容器内监听 `80`，主机只绑定 `127.0.0.1:8080`。HTTPS 由 Tailscale Serve 终止，密码管理服务本身不直接暴露到 `0.0.0.0`。
- `DOMAIN`：客户端最终访问的地址，影响附件、WebSocket 等链接。这里填写 Tailscale Serve 生成的 `https://...ts.net` 地址，必须与实际访问地址一致。
- `ADMIN_TOKEN`：`/admin` 管理页面的登录令牌，从 `.env` 读取，不要写死在 YAML 里。
- `SIGNUPS_ALLOWED`：是否允许自助注册。首次注册完账号后应改为 `"false"`。
- `DATABASE_URL`：连接串中的 `db:5432` 用 Compose 网络里的服务名解析，无需知道容器 IP；密码从 `.env` 插值。
- `depends_on`：等 `db` 的健康检查通过后再启动 Vaultwarden，避免数据库未就绪导致应用启动失败。
- `db` 没有写 `ports`：数据库只在 Compose 内部网络可达，不会暴露到主机。

### 3. 生成管理令牌并启动

先生成两个随机值，分别用于管理面板令牌和数据库密码：

```bash
openssl rand -base64 48     # ADMIN_TOKEN
openssl rand -hex 24        # POSTGRES_PASSWORD
```

复制输出，创建并编辑 `.env`：

```bash
sudo touch .env
sudo nano .env
```

写入两行并保存退出（`=` 两边不要有空格，值不要折行）：

```ini
ADMIN_TOKEN=<粘贴第一条输出>
POSTGRES_PASSWORD=<粘贴第二条输出>
```

最后启动：

```bash
docker compose up -d
docker compose ps
```

> 🔴 文件中的令牌和密码不要提交到 Git，.env 中数据库密码一旦 compose 初始化服务后不要修改，否则后续无法登录，ADMIN_TOKEN可以修改，但是旧令牌可能还会保持有效一段时间，且登录需要新令牌。

### 4. 配置 HTTPS 并首次访问

Vaultwarden 的 Web 前端要求所有 API 请求走 HTTPS，直接用 `http://localhost:8080` 会报 `Insecure URL not allowed. All URLs must use HTTPS.`，无法登录。这里用 Tailscale Serve 终止 TLS，容器仍只监听本机回环。

前置条件：服务器和访问设备都加入同一 tailnet，并在 Tailscale 管理后台的 **DNS** 页面启用 **MagicDNS** 和 **HTTPS Certificates**。

启动 Serve：

```bash
tailscale serve --bg 8080
tailscale serve status
```

输出中的地址形如 `https://<主机名>.<tailnet>.ts.net`。它会把 443 的 HTTPS 流量转发到本机 `127.0.0.1:8080`，证书自动签发和续期，且只在 tailnet 内可见。Serve 配置会持久化，重启后仍然有效；不再需要时用 `tailscale serve reset` 清除。

把 `docker-compose.yaml` 中的 `DOMAIN` 改成这个地址，然后重建：

```bash
docker compose up -d
docker compose ps
```

浏览器打开该 HTTPS 地址，完成第一个账号注册。这个账号拥有全部数据。

1. 注册完成后，把 `SIGNUPS_ALLOWED` 改为 `"false"`，再执行 `docker compose up -d` 生效，数据目录不受影响。
2. 需要添加其他用户时，访问 `https://<主机名>.<tailnet>.ts.net/admin`，粘贴 `.env` 中的 `ADMIN_TOKEN` 登录，通过邀请方式添加。
3. 手机、桌面客户端和浏览器扩展使用同一个 HTTPS 地址，设备需要加入同一 tailnet。不要填写 `http://` 或 `localhost` 地址。
4. Tailscale Serve 只在 tailnet 内提供服务，不会暴露公网。确实需要公网访问时再考虑 Funnel 或带 TLS 的反向代理，并额外做访问控制。

### 5. 备份与升级

Vaultwarden 的附件、密钥与配置在 `/srv/vaultwarden/vw_data`，先停服务再打包，避免写入过程中备份不一致：

```bash
docker compose stop vaultwarden
sudo tar czf "vaultwarden-data-$(date +%F).tar.gz" -C /srv/vaultwarden vw_data
docker compose start vaultwarden
```

PostgreSQL 使用逻辑备份，不要把运行中的 `db_data` 数据目录直接打包：

```bash
docker compose exec -T db pg_dump -U vaultwarden vaultwarden | gzip > vaultwarden-db-$(date +%F).sql.gz
```

升级镜像：

```bash
docker compose pull
docker compose up -d
```

数据目录在宿主机上，重建容器不会影响数据。升级前先完成上面两项备份：Vaultwarden 的数据库结构升级后通常不能回退到旧版本。

### 6. 常用字段

| 字段 | 说明 |
| --- | --- |
| `image` | 使用的镜像，建议固定标签 |
| `build` | 从 Dockerfile 构建，可替代 `image` |
| `container_name` | 固定容器名；不写则默认为 `<项目>-<服务>-<序号>` |
| `ports` | 端口映射，`"主机端口:容器端口"` |
| `volumes` | 目录或命名卷挂载 |
| `environment` | 环境变量（键值或列表） |
| `env_file` | 从文件批量注入环境变量 |
| `restart` | 重启策略，取值同 `docker run --restart` |
| `depends_on` | 启动顺序与依赖就绪条件 |
| `healthcheck` | 健康检查，配合 `depends_on` 的条件使用 |
| `command` | 覆盖镜像默认启动命令 |
| `networks` | 自定义网络 |

### 7. 注意事项

- `ports` 默认绑定 `0.0.0.0`，公网可达。密码管理服务务必写成 `"127.0.0.1:<端口>:80"`，远程访问走 Tailscale Serve 或带 TLS 的反向代理；客户端不接受明文 HTTP。
- `volumes` 里的绝对路径是绑定挂载，需要提前创建并保证可写；也可以写成 `./data` 相对于 Compose 文件目录，或用 `vw_data:/data` 交给 Docker 管理为命名卷。
- 容器路径后的 `:ro` 表示只读挂载，容器内不能修改宿主机文件。
- `depends_on` 默认只保证启动顺序，不保证服务就绪；本示例配合 `condition: service_healthy`，会等 PostgreSQL 健康检查通过。
- 单实例场景可写 `container_name` 方便记忆；需要扩容或多副本时不要写，否则端口和名称会冲突。
- `DOMAIN` 必须与实际使用的 HTTPS 地址（如 Tailscale Serve 生成的 `https://...ts.net`）一致，否则附件下载、WebSocket 通知等链接会指向错误地址。
- `DATABASE_URL` 中的密码需做 URL 编码；用 `openssl rand -hex` 生成可避开特殊字符。
- `postgres:18` 的默认数据目录是 `/var/lib/postgresql/18/docker`，因此卷挂在 `/var/lib/postgresql`；17 及更早版本才是 `/var/lib/postgresql/data`。

## 四、环境变量

两种变量来源经常混淆：

- `.env`：供 Compose **解析配置文件**时做插值，文件默认放在项目目录，无需在 YAML 中声明。
- `env_file`：把文件内容**注入容器**，相当于批量 `--env-file`；它不参与 YAML 插值。

插值支持默认值和必填校验：

```yaml
environment:
  ADMIN_TOKEN: ${ADMIN_TOKEN:?请在 .env 中设置 ADMIN_TOKEN}
  SIGNUPS_ALLOWED: "${SIGNUPS_ALLOWED:-false}"
  TZ: ${TZ:-Asia/Shanghai}
```

`:-` 在变量为空时使用默认值，`:?` 在变量缺失时直接报错并停止。`.env` 常含密码或令牌，不要提交到 Git；仓库里可保留一份 `.env.example` 作为模板。

## 五、常用命令

```bash
docker compose up -d                     # 创建并后台启动
docker compose ps                        # 查看项目内服务状态
docker compose logs -f vaultwarden       # 跟踪指定服务日志
docker compose exec vaultwarden ls /data # 在运行中的容器里执行命令
docker compose restart vaultwarden       # 重启服务
docker compose stop                      # 停止（不删除）
docker compose start                     # 启动已存在的服务
docker compose down                      # 停止并删除容器、网络
docker compose down -v                   # 同时删除命名卷（绑定挂载不受影响）
```

容器生命周期与项目的关系：

- 修改 `compose.yaml` 后再次 `docker compose up -d`，只会重建配置发生变化的服务，其余服务不受影响。
- 使用 `build` 的服务，代码或 Dockerfile 更新后加 `--build`；`image` 拉取新版本则先 `docker compose pull`，再 `up -d`。
- 配置改动没生效时可用 `--force-recreate` 强制重建，但会重启服务，按需使用。
- `docker compose config` 会校验语法并输出插值后的最终配置，排错时优先看它。

> 🔴 本例使用绑定挂载，`docker compose down -v` 不会删除 `/srv/vaultwarden/vw_data` 和 `/srv/vaultwarden/db_data`。但直接删除 `/srv/vaultwarden` 会连数据一起删掉，清理前先确认已备份。

## 六、卷与网络

- 本例使用绑定挂载，数据直接位于 `/srv/vaultwarden/vw_data` 和 `/srv/vaultwarden/db_data`，便于查看、备份和迁移；目录权限需与容器内用户匹配。
- 也可以改用命名卷（如 `vw_data:/data`），由 Docker 统一存放在 `/var/lib/docker/volumes/<项目名>_<卷名>`，不用关心宿主机路径。
- `down` 只删除容器和网络；加 `-v` 仅删除命名卷，绑定挂载的宿主机目录不受影响。
- 同一项目内的服务默认加入同一个网络，可直接用**服务名**互相访问；示例中的 `DATABASE_URL` 就是连接 `db:5432`，不需要发布数据库端口。
- 需要访问项目外容器或固定网段时才声明自定义 `networks`，一般无需手动配置。

## 七、使用建议

- 镜像固定到具体版本，避免 `latest` 在重建时悄悄升级；升级前先备份数据目录和数据库。
- Compose 文件与 `.env.example` 一起提交，实际 `.env` 加入 `.gitignore`。
- 多项目共用一台主机时，用目录区分项目并显式指定 `name:`，避免容器名和卷名冲突。
- 旧教程里的 `docker-compose`（v1）已停止维护，统一使用 `docker compose`。
- 复杂配置先用 `docker compose config` 展开检查，再执行 `up -d`。
- PostgreSQL 大版本升级不能只改镜像标签，需用 `pg_dump` 导出后导入新版本，或按官方 `pg_upgrade` 流程处理。
- 密码管理服务至少做到：强 `ADMIN_TOKEN`、注册关闭、数据目录和数据库定期备份并验证可恢复、对外访问使用 HTTPS。
- 用 Tailscale Serve 时服务只对 tailnet 开放；不要为了省事开启 Funnel，除非明确需要公网访问并已做好访问控制。
