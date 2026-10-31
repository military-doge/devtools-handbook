# Docker Compose 编排

> 官方文档：<https://docs.docker.com/compose/>  
> 配置文件参考：<https://docs.docker.com/reference/compose-file/>  
> 适用于随 `docker-compose-plugin` 安装的 Compose v2，命令为 `docker compose`（无连字符）。  
> 前置：已完成 [Docker 安装与配置](docker_install.md)。

单容器用 `docker run` 足够；一旦涉及多个容器、固定端口与挂载参数、反复启停，手写命令既长又容易漏参数。Compose 把整套服务写进一个 YAML 文件，用一条命令创建、更新、停止和删除，配置可版本化、可复用。

## 一、目录结构与项目名

```text
myapp/
├── compose.yaml
├── .env
└── html/
    └── index.html
```

Compose 默认读取当前目录的 `compose.yaml`（也兼容 `docker-compose.yaml` / `.yml`）。项目名默认取所在目录名，也可用顶层 `name:` 或 `-p` 参数指定；容器、网络、命名卷会自动加上项目名前缀。

> Compose v2 已不需要顶层 `version:` 字段，写上反而会提示过时。

## 二、基础示例

```yaml
name: demo

services:
  web:
    image: nginx:stable
    container_name: demo-web
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:18
    environment:
      POSTGRES_DB: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

运行前先准备挂载目录和密码：

```bash
mkdir -p html
echo '<h1>Hello</h1>' > html/index.html

cat > .env <<EOF
POSTGRES_PASSWORD=change-me
EOF

docker compose up -d
```

浏览器打开 `http://127.0.0.1:8080` 即可访问；Postgres 只在 Compose 内部网络暴露，不映射到主机。

常用字段：

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

几点容易踩坑的地方：

- `ports` 默认绑定 `0.0.0.0`，公网可达。只给本机或 SSH 隧道用时写成 `"127.0.0.1:8080:80"`。
- `volumes` 里 `./html` 是相对 `compose.yaml` 所在目录的路径，需要提前创建；`db_data` 这种写法是命名卷，由 Docker 管理。
- 容器路径后的 `:ro` 表示只读挂载，容器内不能修改宿主机文件。
- `depends_on` 默认只保证启动顺序，不保证服务就绪；配合 `condition: service_healthy` 才会等健康检查通过。
- 单实例场景可写 `container_name` 方便记忆；需要扩容或多副本时不要写，否则端口和名称会冲突。

## 三、环境变量

两种变量来源经常混淆：

- `.env`：供 Compose **解析配置文件**时做插值，文件默认放在项目目录，无需在 YAML 中声明。
- `env_file`：把文件内容**注入容器**，相当于批量 `--env-file`；它不参与 YAML 插值。

插值支持默认值和必填校验：

```yaml
environment:
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?请在 .env 中设置 POSTGRES_PASSWORD}
  TZ: ${TZ:-Asia/Shanghai}
```

`:-` 在变量为空时使用默认值，`:?` 在变量缺失时直接报错并停止。`.env` 常含密码，不要提交到 Git；仓库里可保留一份 `.env.example` 作为模板。

## 四、常用命令

```bash
docker compose up -d              # 创建并后台启动
docker compose ps                 # 查看项目内服务状态
docker compose logs -f web        # 跟踪指定服务日志
docker compose exec db psql -U postgres   # 在运行中的容器里执行命令
docker compose restart web        # 重启服务
docker compose stop               # 停止（不删除）
docker compose start              # 启动已存在的服务
docker compose down               # 停止并删除容器、网络
docker compose down -v            # 同时删除命名卷
```

容器生命周期与项目的关系：

- 修改 `compose.yaml` 后再次 `docker compose up -d`，只会重建配置发生变化的服务，其余服务不受影响。
- 使用 `build` 的服务，代码或 Dockerfile 更新后加 `--build`；`image` 拉取新版本则先 `docker compose pull`，再 `up -d`。
- 配置改动没生效时可用 `--force-recreate` 强制重建，但会重启服务，按需使用。
- `docker compose config` 会校验语法并输出插值后的最终配置，排错时优先看它。

> 🔴 `docker compose down -v` 会删除命名卷，数据库等数据随之丢失。日常停止用 `stop`，确需清理再用 `down`；删除卷前先确认已备份。

## 五、卷与网络

- 命名卷（如 `db_data`）实际存放在 `/var/lib/docker/volumes/<项目名>_db_data`，由 Docker 管理，适合数据库等持久化数据。
- 绑定挂载（如 `./html:/usr/share/nginx/html`）直接使用主机目录，适合配置文件和需要随时改动的代码，`down` 不会删除主机目录。
- `down` 默认保留命名卷，只有加 `-v` 才删除；绑定挂载不受 `-v` 影响。
- 同一项目内的服务默认加入同一个网络，可直接用**服务名**互相访问，例如应用里把数据库地址写成 `db:5432`。
- 需要访问项目外容器或固定网段时才声明自定义 `networks`，一般无需手动配置。

## 六、示例：用 Compose 部署 Portainer

把 [Docker 安装与配置](docker_install.md) 中的 `docker run` 改写成 Compose：

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "127.0.0.1:9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

```bash
docker compose up -d
```

效果与手动 `docker run` 相同，但端口、挂载和重启策略都记录在文件里，换机器或重装时直接复制目录即可。面板的初始化、setup token 与 SSH 隧道访问方式见原章节。

## 七、使用建议

- 镜像固定到具体版本，避免 `latest` 在重建时悄悄升级。
- Compose 文件与 `.env.example` 一起提交，实际 `.env` 加入 `.gitignore`。
- 多项目共用一台主机时，用目录区分项目并显式指定 `name:`，避免容器名和卷名冲突。
- 旧教程里的 `docker-compose`（v1）已停止维护，统一使用 `docker compose`。
- 复杂配置先用 `docker compose config` 展开检查，再执行 `up -d`。
