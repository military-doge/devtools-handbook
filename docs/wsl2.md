# WSL2 安装与配置

> 官方文档：<https://learn.microsoft.com/windows/wsl/>  
> 适用于 Windows 10 2004（Build 19041）及以上 / Windows 11  
> 本文档默认只维护一个发行版（Ubuntu-24.04），多环境隔离交给容器或 devcontainer 处理。

WSL2 在 Windows 上运行完整的 Linux 内核，与 Windows 的文件、终端、IDE 互通。对已安装 Windows 的机器，既能保留日常使用的便利，又能提供完整的 Linux 开发环境。以下命令除特别说明外均在 Windows PowerShell 中执行。

## 一、前置检查

- 系统版本满足要求
- 主板 BIOS/UEFI 已开启虚拟化（Intel VT-x / AMD-V）

在任务管理器「性能 → CPU」中确认「虚拟化：已启用」即可。

## 二、安装 WSL 本体以及 Linux 发行版

查看在线可安装的发行版：

```powershell
wsl --list --online  # 报错请自行解决网络环境问题
```

安装指定发行版到指定盘符目录：

```powershell
mkdir D:\WSL                                        # 创建目标目录
wsl --install -d Ubuntu-24.04 --location D:\WSL     # 安装指定发行版到该目录
```

该命令会启用虚拟机平台、安装 WSL2 与所选发行版。完成后重启系统，首次启动时设置 Linux 用户名和密码。

`--location` 需 Microsoft Store 版 WSL；不支持时先按默认位置安装，再用第六节的 `wsl --manage ... --move` 迁移。

若系统较旧、提示不支持 `--install`，先手动启用两个功能并重启：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重启后再执行上文的安装命令。

## 三、常用操作

```powershell
wsl                            # 进入默认发行版
wsl -l -v                      # 列出发行版与运行状态
wsl --shutdown                 # 关闭所有发行版与虚拟机
```

文件互通：

- Windows 访问 Linux：资源管理器地址栏输入 `\\wsl.localhost`
- Linux 访问 Windows：`/mnt/c/Users/<用户名>`

`/mnt/c` 经 Windows 主机文件共享挂载而来，读写带宽和延迟都明显劣于 Linux 原生文件系统。

## 四、基础配置 /etc/wsl.conf

WSL2 默认用自带的精简 init 启动，它只管挂载 Windows 盘和打通互操作，不负责服务管理：`systemctl` 不可用，`apt install` 的 sshd、docker 等守护进程也不会开机自启。开启 `[boot] systemd=true` 后由 systemd 作为 PID 1 接管 init，服务的自启、重启、`journalctl` 日志、cgroup 等行为才和普通 Linux 一致。

在 WSL 内编辑 `/etc/wsl.conf`，修改后执行 `wsl --shutdown` 再进入生效：

```ini
[boot]
systemd=true

[user]
default=<用户名>
```

- `[boot] systemd=true`：Ubuntu 23.04 起（含 24.04）官方镜像已默认开启，其他发行版或旧镜像需手动加
- `[user] default=<用户名>`：指定默认登录用户。正常安装会自动写入，用 `--import` 导入的发行版会退回 root，需要在这里补

## 五、资源与网络 .wslconfig

Windows 侧文件：`C:\Users\<用户名>\.wslconfig`，修改后 `wsl --shutdown` 生效：

```ini
[wsl2]
memory=8GB                 # 主机 32 GB 内存子系统内存可以加到 16 GB
processors=8               # 限制最多使用的逻辑处理器数（含超线程）
swap=2GB                   # 交换空间，Linux 内存吃紧时将不活跃的内存页换出到 swap
networkingMode=mirrored    # 镜像模式，让 WSL 与 Windows 共享网络接口
```

- `networkingMode=mirrored`：主要作用是把 Windows 的网络环境带进 WSL，全局代理（含 TUN 模式）、DNS 等配置可直接沿用，不必在 Linux 内重复设置。
- **避免服务暴露到外部**：mirrored 模式下，服务保持监听 `127.0.0.1`（而非 `0.0.0.0` / `::`），避免其出现在 Windows 的网卡上，被外部设备访问；Docker 发布端口时用 `-p 127.0.0.1:<端口>:<容器端口>`，不要用 `-p <端口>:<容器端口>`，后者默认绑定 `0.0.0.0`。Windows 防火墙默认阻止入站连接，可作第二道防线，不要为该端口添加放行规则。

## 六、迁移到其他磁盘

新版 WSL 可直接移动发行版，无需导出：

```powershell
wsl --shutdown
wsl --manage Ubuntu-24.04 --move D:\WSL
```

旧版本可用导出 / 导入：

```powershell
wsl --export Ubuntu-24.04 D:\WSL\ubuntu.tar
wsl --import Ubuntu-24.04 D:\WSL D:\WSL\ubuntu.tar --version 2
# wsl --unregister Ubuntu-24.04
```

导入后默认用户会变回 root，需在 `/etc/wsl.conf` 中补 `[user] default=<用户名>`。

> 🔴 `wsl --unregister` 会删除该发行版的全部数据，确认导入正常后再删除。  
> 定时导出备份可以用于系统恢复

## 七、卸载 Linux 发行版

```powershell
wsl --unregister Ubuntu-24.04
```
