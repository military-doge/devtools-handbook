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

把公钥内容放进服务器的 `~/.ssh/authorized_keys`，每个公钥单独一行，格式和 `.pub` 文件一致：

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... me@laptop
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... work-pc
```

公钥如何传到服务器视具体环境而定，始终只上传 `.pub` 公钥，私钥不要离开本机。

## 三、客户端配置

编辑 `~/.ssh/config`（Windows 为 `C:\Users\<用户名>\.ssh\config`）：

```text
Host <myserver>
    HostName <受控端IP>
    User <受控端用户名>
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

配置好后，以下两种方式等效：

```bash
ssh <受控端用户名>@<受控端IP>
ssh <myserver>
```

多台机器各写一段 `Host`。公共选项可用通配段统一设置，放在文件末尾：

```text
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

`ServerAliveInterval` 每 60 秒发一次心跳，连续 3 次无响应才断开，避免空闲掉线。

## 四、VSCode 远程开发

VSCode 通过 `Remote - SSH` 扩展直接复用上一步的 `~/.ssh/config`，主机、用户、端口、私钥都不必在插件里重复填写。

1. 控制端在扩展市场安装 `Remote - SSH`（`ms-vscode-remote.remote-ssh`）。
2. 按 `Ctrl+Shift+P` 打开命令面板，执行 `Remote-SSH: Connect to Host`，选择 `config` 中的 `Host` 别名（如 `<myserver>`）。也可点击左下角的远程指示器选择连接到主机。
3. 首次连接会询问远端平台，按服务器实际系统选择（通常为 `Linux`），VSCode 自动在服务器安装服务端组件。
4. 连接成功后打开远端目录，之后的文件编辑、终端、调试都在服务器上执行。

## 五、端口转发

有些服务只在服务器内部监听 `127.0.0.1:<远端端口>`，不对公网开放。

SSH 本地转发可以把该端口映射到本机，每次执行 `ssh` 时手动带上转发参数：

```bash
ssh -L <本地端口>:localhost:<远端端口> <myserver>
```

常用做法是写进 `~/.ssh/config`（Windows 为 `C:\Users\<用户名>\.ssh\config`）：

```text
Host <myserver>
    HostName <受控端IP>
    User <受控端用户名>
    LocalForward <本地端口> localhost:<远端端口>
```

之后执行 `ssh <myserver>`，会自动带上主机、用户和转发参数。连接保持期间，在控制端用浏览器或其他客户端访问 `localhost:<本地端口>` 即可。

## 六、服务端配置

编辑 `/etc/ssh/sshd_config`。

常用项：

```text
Port 22                              # SSH 监听端口
PermitRootLogin prohibit-password    # root 禁止密码登录，仅允许公钥
PasswordAuthentication no            # 禁用密码认证
PubkeyAuthentication yes             # 启用公钥认证
```

> 🔴 提示：禁用密码登录前，务必先确认公钥已能登录。

修改后先校验再重启：

```bash
sudo sshd -t

# Debian / Ubuntu
sudo systemctl restart ssh

# RHEL / CentOS
sudo systemctl restart sshd
```
