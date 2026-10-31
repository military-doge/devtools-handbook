# Docker 镜像

> 官方 CLI 参考：<https://docs.docker.com/reference/cli/docker/image/>
> 镜像的日常使用见 [Docker 容器操作](docker_containers.md)。

镜像是只读模板，容器是它的运行实例。镜像名通常写成 `仓库/名称:标签`，省略标签时默认 `latest`：

```text
debian:12
library/nginx:stable
ghcr.io/org/app:v1.2.0
```

日常使用建议固定具体版本，避免 `latest` 在重建容器时带来不可预期的变化。

## 一、拉取与查看

```bash
docker pull debian:12
docker pull nginx:stable

docker images              # 本地镜像列表
docker image inspect debian:12   # 查看镜像详细信息
docker history debian:12         # 查看镜像分层构建历史
```

在 Docker Hub 查找镜像与标签，例如 Debian：<https://hub.docker.com/_/debian>。在页面 **Tags** 中选择所需版本。

`docker pull` 只下载镜像，不会创建容器；`docker images` 中同一镜像的不同标签会分成多行，镜像 ID 可能相同。

## 二、打标签

给已有镜像起别名，便于在 Compose 或本地构建流程中复用：

```bash
docker tag debian:12 myrepo/debian:base
```

`docker tag` 不复制镜像数据，只是新增一个引用。删除其中一个标签不影响其他标签。

## 三、删除镜像

```bash
docker rmi debian:12
```

镜像被容器占用时需先删除容器，或加 `-f` 强制删除。`-f` 只强制删除镜像，不会删除关联容器；被强制删除后，容器会保留但镜像引用变成 `<none>`。

清理未被任何容器使用的镜像：

```bash
docker image prune        # 仅悬空镜像（<none>）
docker image prune -a     # 所有未被容器使用的镜像
```

`-a` 会删除本地所有闲置镜像，再次使用需要重新拉取；执行前先确认没有离线环境依赖。

## 四、导出与导入

无网络环境或需要迁移镜像时，可将镜像保存为文件：

```bash
docker save -o debian12.tar debian:12
docker load -i debian12.tar
```

导出的 tar 包含完整镜像层，文件通常较大；跨机器传输后 `docker load` 即可恢复，无需重新拉取。

## 五、构建自定义镜像

镜像可以在已有镜像基础上，通过 `Dockerfile` 构建。最小示例：

```dockerfile
FROM nginx:stable
COPY html/ /usr/share/nginx/html/
```

```bash
docker build -t my-nginx:v1 .
docker run -d --name web -p 8080:80 my-nginx:v1
```

`-t` 指定镜像名与标签，末尾的 `.` 是构建上下文目录。构建后的镜像与拉取的镜像用法完全一致，也可被 Compose 直接引用。
