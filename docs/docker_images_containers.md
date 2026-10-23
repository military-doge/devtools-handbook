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

镜像被容器占用时需先删除容器，或加 `-f` 强制删除。`-f` 只强制删除镜像，不会删除关联容器；被强制删除后，容器会保留但镜像引用变成 `<none>`。

## 二、管理容器

### 1. 创建实例

交互式容器（退出后容器停止）：

```bash
docker run -it --name debian-demo debian:12 bash
```

镜像名后面的 `bash` 是容器启动时执行的命令，会覆盖镜像默认命令；`bash` 与 `/bin/bash` 等价。

后台运行并映射端口：

```bash
docker run -d --name web -p 8080:80 nginx
```

挂载主机目录到容器（把主机的 `/home/user/data` 映射到容器 `/data`）：

```bash
docker run -it --name debian-mount -v /home/user/data:/data debian:12 bash
```

以上示例会分别创建 `debian-demo`、`web`、`debian-mount` 三个容器。

`-v 主机路径:容器路径` 表示把主机目录挂载到容器：冒号左边必须是主机上**已存在**的目录；冒号右边的容器路径由 Docker 在实例化时**自动创建**。两者指向同一份数据，任一侧修改，另一侧立即可见。

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

`--restart` 决定容器退出或 Docker 服务重启后是否自动拉起：

- `no`（默认）：不自动重启
- `on-failure[:次数]`：异常退出（退出码非 0）时重启，可限制次数，如 `on-failure:3`
- `always`：总是自动重启，Docker 服务重启后也会拉起
- `unless-stopped`：同 `always`，但手动 `docker stop` 后不再自动启动（日常推荐）

已存在的容器可用 `docker update --restart=<策略> <实例名>` 修改。

宿主机重启后的自动拉起，依赖 Docker 守护进程开机自启（`systemctl enable docker`）。

### 2. 查看状态

```bash
docker ps                 # 运行中的容器
docker ps -a              # 所有容器（含已停止）
docker inspect <实例名>    # 详细信息（IP、挂载、配置）
docker logs <实例名>       # 容器日志
docker stats              # 实时资源占用
```

### 3. 启动 / 停止 / 重启

```bash
docker stop <实例名>
docker start <实例名>
docker restart <实例名>
```

`start` 只启动已存在但停止的容器，不能创建新容器。

### 4. 进入运行中的容器

`docker exec` 与 `docker attach` 都只对运行中的容器有效。

在容器内新开一个 shell（推荐）：

```bash
docker exec -it <实例名> bash
```

`exit` 退出的是这个新开的 shell，容器本身继续运行。

也可以不开 shell，直接在容器里执行一条命令（俗称「穿透执行」）：

```bash
docker exec <实例名> cat /etc/os-release   # 查看系统版本
docker exec <实例名> ls /etc               # 列目录
docker exec -u root <实例名> whoami        # 以 root 身份执行
```

这种执行方式不需要 `-it`；只有命令本身需要交互（如 `bash`、`vim`）时才加。

#### attach 与 exec 的区别

容器里 PID 1 的进程决定容器的生死。`docker attach` 接入的正是这个主进程：

```bash
docker attach <实例名>
```

- `attach`：你的键盘直接接到主进程上。`exit` 或 `Ctrl+C` 会结束主进程，容器随之停止；`Ctrl+P` `Ctrl+Q` 可脱离而容器继续运行。主进程需以 `-it` 启动、且会读取 stdin，交互才方便。
- `exec`：在容器内另开一个进程，与主进程互不影响。`exit` 只关掉这个新进程，容器照常运行，日常排查更推荐。

一句话：`attach` 是接管主进程，`exec` 是另开一个窗口。

### 5. 删除容器

```bash
docker rm <实例名>       # 删除已停止的容器
docker rm -f <实例名>    # 强制删除运行中的容器
```

## 三、完整示例（Debian）

```bash
# 拉取镜像
docker pull debian:12

# 后台以交互模式运行，bash 保持前台，容器常驻
docker run -dit --name debian-run debian:12 bash

# 查看状态
docker ps

# 进入容器
docker exec -it debian-run bash

# 停止、启动
docker stop debian-run
docker start debian-run

# 删除容器与镜像
docker rm -f debian-run
docker rmi debian:12
```

`-dit` 中 `-d` 让容器后台运行，`-it` 保留交互终端，bash 一直等待输入，容器因此保持运行；若只写 `-d` 执行一次性命令，容器会立即退出。
