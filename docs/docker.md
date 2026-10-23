# Docker Engine 安装与配置

> 官方文档：<https://docs.docker.com/engine/install/ubuntu/>
> 适用于 Ubuntu 26.10 / 26.04 LTS / 24.04 LTS / 22.04 LTS（64 位）

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

> 安装特定版本：先 `apt list --all-versions docker-ce` 查版本，再
> `sudo apt install docker-ce=<版本> docker-ce-cli=<版本> containerd.io docker-buildx-plugin docker-compose-plugin`

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

重新登录（或 `newgrp docker`）后生效。

> 安全提示：`docker` 组≈root 权限，仅授予可信用户。

### 2. 配置镜像加速

编辑 `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ]
}
```

然后重启：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

用 `docker info` 查看 `Registry Mirrors` 是否生效。

> 注意：上述为第三方镜像，可用性随时间变化，失效时自行替换。

### 3. 日志轮转

在 `/etc/docker/daemon.json` 中追加（与镜像加速合并到同一 JSON）：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

重启 docker 生效。仅对**新建**容器生效，旧容器需重建。

### 4. 更改数据目录（可选）

默认数据在 `/var/lib/docker`。迁移前先停止 docker，再把数据整体搬到新目录，然后：

```json
{
  "data-root": "/data/docker"
}
```

重启 docker 并用 `docker info | grep "Docker Root Dir"` 确认。

### 5. 常用服务管理

```bash
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl enable docker    # 开机自启
sudo systemctl status docker
```

## 六、卸载

```bash
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm /etc/apt/sources.list.d/docker.sources
sudo rm /etc/apt/keyrings/docker.asc
```

镜像、容器、卷及自定义配置不会自动删除，需手动清理。
