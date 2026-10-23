# Docker 镜像与容器操作

> 官方 CLI 参考：<https://docs.docker.com/reference/cli/docker/>

## 一、管理镜像

### 1. 拉取镜像

```bash
docker pull debian:12
docker images          # 查看本地已下载的镜像
```

`docker pull debian:12` 拉取 Debian 12（bookworm）镜像；`docker images` 查看本地已有镜像。

在 Docker Hub 查找镜像与标签，例如 Debian：<https://hub.docker.com/_/debian>。在页面 **Tags** 中选择所需版本。

### 2. 删除镜像

```bash
docker rmi debian:12
```

镜像被容器占用时需先删除容器，或加 `-f` 强制删除。

## 二、管理容器

### 1. 创建实例

交互式容器（退出后容器停止）：

```bash
docker run -it --name debian-demo debian:12 bash
```

后台运行并映射端口：

```bash
docker run -d --name web -p 8080:80 nginx
```

常用参数：

| 参数 | 说明 |
| --- | --- |
| `-i` | 保持标准输入 |
| `-t` | 分配伪终端，`-it` 一起用 |
| `-d` | 后台运行 |
| `--name` | 指定容器名 |
| `-p 主机端口:容器端口` | 端口映射 |
| `-v 主机路径:容器路径` | 目录挂载 |
| `--restart` | 重启策略，如 `always` |

### 2. 查看状态

```bash
docker ps                 # 运行中的容器
docker ps -a              # 所有容器（含已停止）
docker inspect web        # 详细信息（IP、挂载、配置）
docker logs web           # 容器日志
docker stats              # 实时资源占用
```

### 3. 启动 / 停止 / 重启

```bash
docker stop web
docker start web
docker restart web
```

`start` 只启动已存在但停止的容器，不能创建新容器。

### 4. 进入运行中的容器

```bash
docker exec -it debian-demo bash
```

### 5. 删除容器

```bash
docker rm web          # 删除已停止的容器
docker rm -f web       # 强制删除运行中的容器
```

## 三、完整示例（Debian）

```bash
# 拉取镜像
docker pull debian:12

# 后台启动一个保持运行的容器
docker run -d --name debian-sleeper debian:12 sleep infinity

# 查看状态
docker ps

# 进入容器
docker exec -it debian-sleeper bash

# 停止、启动
docker stop debian-sleeper
docker start debian-sleeper

# 删除容器与镜像
docker rm -f debian-sleeper
docker rmi debian:12
```

Debian 镜像默认执行完命令就退出，所以后台运行时需要一个持续命令（如 `sleep infinity`），否则 `docker ps` 看不到它。
