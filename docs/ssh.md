# SSH 配置与免密登录

> 官方手册：<https://man.openbsd.org/ssh_config>
> 适用于 Linux / macOS / Windows（WSL）

## 前置：Tailscale 组网

> 官方文档：<https://tailscale.com/kb/1017/install>

服务器没有公网 IP、或不想向公网暴露 22 端口时，先用 Tailscale 把设备接入同一个虚拟内网。每台设备获得一个 `100.x.y.z` 地址，彼此可直连，流量端到端加密。

控制端通常是 Windows / macOS 这类图形系统，安装官方客户端，注册账号并授权设备即可。

Linux 服务器安装并登录：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

`tailscale up` 会输出一个授权地址并等待确认，例如：

```text
To authenticate, visit:
    https://login.tailscale.com/a/xxxxxxxxxxxx
```

在任意设备的浏览器里打开该地址，用同一账号登录并授权，服务器即加入同一 tailnet。

查看本机地址与在线设备：

```bash
tailscale ip -4
tailscale status
```

地址可以自选：在管理后台的设备详情里修改 IPv4 地址，换成一个好记的 `100.x.y.z` 即可。

之后直接通过虚拟地址 SSH，无需端口映射或公网 IP：

```bash
ssh user@100.x.y.z
```

启用 MagicDNS 后可用设备名代替地址：

```bash
ssh user@hostname
```

也可以启用 Tailscale SSH，由 Tailscale 校验身份，免去公钥分发：

```bash
sudo tailscale up --ssh
```

> 🔴 安全提示：若怀疑任意一台已加入虚拟局域网的设备遭到入侵，除切断其网络连接外，也可在手机端登录 Tailscale 控制台，将该设备踢下线，立即阻断它对内网的访问。

## 一、生成密钥

密钥在控制端（你自己的电脑，通常是 Windows / macOS）生成，私钥留在本机，公钥交给服务器：

```bash
ssh-keygen -t ed25519 -C "备注"
```

一路回车即可，默认生成：

```text
Linux / macOS
~/.ssh/id_ed25519      # 私钥，留在本机
~/.ssh/id_ed25519.pub  # 公钥，交给服务器

Windows
C:\Users\<用户名>\.ssh\id_ed25519      # 私钥，留在本机
C:\Users\<用户名>\.ssh\id_ed25519.pub  # 公钥，交给服务器
```

## 二、上传公钥

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
```

非默认端口：

```bash
ssh-copy-id -p 2222 -i ~/.ssh/id_ed25519.pub user@host
```

服务器没有 `ssh-copy-id` 时手动追加：

```bash
cat ~/.ssh/id_ed25519.pub | ssh -p 2222 user@host \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

始终只上传 `.pub` 公钥，私钥不要离开本机。

## 三、客户端配置

编辑 `~/.ssh/config`（Windows 为 `C:\Users\<用户名>\.ssh\config`）：

```text
Host myserver
    HostName 192.0.2.10
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

之后 `ssh myserver` 等价于：

```bash
ssh -p 22 -i ~/.ssh/id_ed25519 deploy@192.0.2.10
```

多台机器各写一段 `Host`。公共选项可用通配段统一设置，放在文件末尾：

```text
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

`ServerAliveInterval` 每 60 秒发一次心跳，连续 3 次无响应才断开，避免空闲掉线。

## 四、端口转发

### 1. 本地转发

把远端服务映射到本机端口，例如远端仅监听 `127.0.0.1:9443` 的面板：

```bash
ssh -L 9443:localhost:9443 myserver
```

浏览器访问 `https://localhost:9443`。写入配置后 `ssh myserver` 自动建立：

```text
Host myserver
    LocalForward 9443 localhost:9443
```

### 2. 远端转发

反向：把本机服务暴露到远端。

```bash
ssh -R 8080:localhost:80 myserver
```

远端 `localhost:8080` 的请求转发到本机 80 端口。默认只绑定远端的回环地址，不对外开放。

### 3. 动态转发

在本机开一个 SOCKS5 代理：

```bash
ssh -D 1080 myserver
```

浏览器或系统代理填 `127.0.0.1:1080`，流量经 SSH 从远端出口发出。它只提供代理，不是全局 VPN。

## 五、服务端配置

主配置为 `/etc/ssh/sshd_config`；Debian / Ubuntu 默认还会加载 `/etc/ssh/sshd_config.d/*.conf`，可把改动放进独立片段。

常用项：

```text
Port 22
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
```

修改后先校验再重启：

```bash
sudo sshd -t
sudo systemctl restart ssh
```

- 禁用密码登录前，先确认公钥已能登录。
- 改端口后旧连接不受影响，用新端口验证成功再关闭当前会话。
- 服务名：Debian / Ubuntu 为 `ssh`，RHEL / CentOS 为 `sshd`。

## 六、ssh-agent

私钥设置了 passphrase 时，用 agent 避免每次输入：

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

部分桌面环境会随登录自动启动 agent，若未运行则手动执行上面的命令。

## 七、常见问题

| 现象 | 原因与处理 |
| --- | --- |
| `Permissions 0755 ... are too open` | 私钥权限过宽：`chmod 600 ~/.ssh/id_ed25519` |
| 仍提示输入密码 | `ssh -v myserver` 查看是否尝试公钥；确认服务端 `PubkeyAuthentication yes`、`authorized_keys` 权限 600、家目录不被组或其他用户写入 |
| `Permission denied (publickey)` | 公钥未上传或传到了错误用户；重新 `ssh-copy-id` |
| `Host key verification failed` | 主机指纹变化：`ssh-keygen -R <host>`（非默认端口用 `ssh-keygen -R "[host]:2222"`），重连时核对指纹 |
| 空闲后连接断开 | 客户端 `ServerAliveInterval`，或服务端 `ClientAliveInterval 60` |
| `ssh 100.x.y.z` 超时 | `tailscale status` 确认对端在线；检查 ACL 是否放行；未直连时会走 DERP 中继，通常仍可连接 |
| 改配置后失联 | 保留旧会话，另开窗口验证新配置再关闭旧会话 |
