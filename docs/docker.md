# Docker Engine 安装与配置

> 官方文档：<https://docs.docker.com/engine/install/ubuntu/>
> 适用于 Ubuntu 26.10 / 26.04 LTS / 24.04 LTS / 22.04 LTS

## 一、卸载冲突包

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
```

## 二、配置官方 apt 源

```bash
# 添加 Docker 官方 GPG key
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 写入 apt 源（自动匹配当前 Ubuntu codename 与架构）
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

## 三、安装 Docker Engine

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 四、验证安装

```bash
sudo systemctl status docker
sudo docker run hello-world
```

## 五、配置

### 1. 免 sudo 使用 docker

```bash
sudo usermod -aG docker $USER
```

重新登录后生效：断开 SSH 再重新连接即可。组成员身份在登录时读取，当前已打开的终端不会自动更新。

验证：

```bash
docker version
```

正常输出同时包含 Client 与 Server 两部分（版本号可能不同）：

```text
Client: Docker Engine - Community
 Version:           29.9.2
 API version:       1.56
 Go version:        go1.26.10
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          29.9.2
  API version:      1.56 (minimum version 1.44)
  Go version:        go1.26.10
  OS/Arch:           linux/amd64
 containerd:
  Version:          2.1.5
 runc:
  Version:          1.3.3
```

若只显示 Client 并报 `permission denied while trying to connect to the docker API`，说明尚未重新登录或未加入 docker 组。

> 安全提示：`docker` 组≈root 权限，仅授予可信用户。

### 2. 开机自启

```bash
sudo systemctl enable docker
```

设置后，系统开机时自动启动 Docker 守护进程，无需手动 `sudo systemctl start docker`，配置了 `restart` 策略的容器也会随之恢复运行。

### 3. 常用服务管理

```bash
sudo systemctl start docker     # 启动
sudo systemctl stop docker      # 停止
sudo systemctl status docker    # 查看状态
```

## 六、图形面板

无头主机没有桌面，图形面板通过浏览器远程管理容器，能显著提升日常效率。这里选用 Portainer（官方文档：<https://docs.portainer.io/start/install-ce/server/docker/linux>）。

### 部署

```bash
docker volume create portainer_data
docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

浏览器打开 `https://<主机IP>:9443`，首次访问设置管理员密码。

> 安全提示：挂载 `/var/run/docker.sock` 等价于 root 权限；不要将面板端口直接暴露公网。

### 更安全的访问方式

仅监听本机，不向网络暴露端口：

```bash
docker run -d -p 127.0.0.1:9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

在控制端的 `~/.ssh/config` 中添加：

```text
Host <服务器IP>
    User <用户名>
    LocalForward 9443 localhost:9443
```

之后执行 `ssh <用户名>@<服务器IP>` 即可，隧道自动带上。浏览器打开 `https://localhost:9443`。

`<服务器IP>`、`<用户名>` 替换为实际值。

控制端 `localhost:9443` 的流量经 SSH 加密转发到主机的回环地址，访问权限等同 SSH 登录权限。

## 七、卸载

```bash
# 卸载 Docker 相关软件包
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
# 删除镜像、容器、卷等数据
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
# 删除 apt 源与 GPG key
sudo rm /etc/apt/sources.list.d/docker.sources
sudo rm /etc/apt/keyrings/docker.asc
```

镜像、容器、卷及自定义配置不会自动删除，需手动清理。
