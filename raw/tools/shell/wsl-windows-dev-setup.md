---
title: "WSL2 与 Windows 开发环境搭建"
date: 2026-06-02
tags: [wsl, wsl2, windows, scoop, nvm, node, ai, development-environment]
type: tool
aliases: [WSL配置, Windows开发环境, WSL2教程, 开发环境搭建]
description: WSL2 与 Windows 开发环境完整搭建指南：WSL2 安装/迁移/内存限制/代理配置/网络互操作、WSL 命令速查表、Scoop 包管理器（安装/bucket/常用软件）、Windows 常用软件清单（开发工具/终端/字体）、NVM Node 版本管理（Windows+Linux 双平台）、AI 开发环境搭建(Claude Code/CCSwitch CLI/代理配置)、Linux 补充安装（dotnet/SSH/常用软件一键脚本）、疑难问题排查
---

# WSL2 与 Windows 开发环境搭建

## WSL2 安装指南

### 安装 Linux 子系统

```powershell
# 安装 WSL（自动启用虚拟平台，首次需重启）
wsl --install

# 查看可安装发行版
wsl --list --online

# 安装指定发行版（如 Ubuntu）
wsl --install Ubuntu

# 列出已安装
wsl -l -v
```

### 设置默认 WSL 版本

```powershell
# 设置 WSL 2 为默认版本
wsl --set-default-version 2

# 将现有发行版转换为 WSL2
wsl --set-version Ubuntu 2
```

### 设置默认用户

```powershell
# 在发行版配置文件中设置
# 例如 Ubuntu：创建 /etc/wsl.conf
```

```ini
[user]
default=username
```

## WSL 常用命令速查表

### 安装与管理

| 命令 | 说明 |
|------|------|
| `wsl --install` | 安装 WSL（需管理员权限） |
| `wsl --list --online` | 查看可安装的 Linux 发行版 |
| `wsl --install <name>` | 安装指定发行版 |
| `wsl --list -v` | 列出已安装的发行版及状态 |
| `wsl --set-version <name> 2` | 转换到 WSL2 |
| `wsl --update` | 更新 WSL 内核 |
| `wsl --shutdown` | 关闭所有 WSL 实例 |
| `wsl --status` | 查看 WSL 状态 |

### 实例操作

| 命令 | 说明 |
|------|------|
| `wsl -d <name>` | 启动指定发行版 |
| `wsl -t <name>` | 终止指定发行版 |
| `wsl --unregister <name>` | 注销并删除发行版 |
| `wsl -e bash -c "cmd"` | 不进入 shell 直接执行命令 |
| `wsl -u <user>` | 以指定用户启动 |

### 导入与导出

| 命令 | 说明 |
|------|------|
| `wsl --export <name> <file>` | 导出为 tar 文件 |
| `wsl --import <name> <path> <file>` | 从 tar 导入到指定目录 |
| `wsl --import <name> <path> <file> --version 2` | 导入为 WSL2 |

### 子系统内命令

```bash
ip addr                  # 查看 IP 地址
uname -a                 # 系统内核信息
df -h                    # 磁盘使用
free -h                  # 内存使用
lsblk                    # 查看挂载的磁盘
mount                    # 查看挂载点
cat /etc/resolv.conf     # DNS 配置
explorer.exe .           # 在 Windows 资源管理器中打开当前目录
code .                   # 在 VSCode 中打开当前目录（需安装 Remote-WSL 扩展）
```

## 迁移 WSL 到非系统盘

避免 WSL 占用 C 盘空间：

```powershell
# ① 关闭所有 WSL 实例
wsl --shutdown

# ② 导出当前发行版为 tar
wsl --export Ubuntu D:\ubuntu.tar

# ③ 注销原发行版
wsl --unregister Ubuntu

# ④ 在目标盘创建目录
mkdir E:\WSL\Ubuntu

# ⑤ 导入到新位置
wsl --import Ubuntu E:\WSL\Ubuntu D:\ubuntu.tar --version 2

# ⑥ 删除临时文件
del D:\ubuntu.tar

# ⑦ 验证结果
wsl -l -v
```

> 导入后默认以 root 登录，需配置默认用户（见上文）。

## WSL 内存与性能配置

### .wslconfig 文件

创建 `%USERPROFILE%\.wslconfig`（Windows 用户目录）：

```text
[wsl2]
memory=8GB
processors=4
swap=2GB
localhostForwarding=true

[experimental]
autoMemoryReclaim=gradual
sparseVhd=true
```

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `memory` | WSL2 最大内存占用 | 总内存的 50% |
| `processors` | 最大 CPU 核心数 | 4 |
| `swap` | 交换空间大小 | 2GB |
| `localhostForwarding` | 允许从 Windows 访问 WSL 服务 | true |
| `autoMemoryReclaim` | 自动回收空闲内存 | gradual |
| `sparseVhd` | 自动收缩虚拟磁盘 | true |

保存后执行 `wsl --shutdown` 重启生效。

## WSL 代理配置

WSL2 中访问 Windows 宿主机代理：

```bash
# 获取 Windows 主机 IP（WSL 网关地址）
export WIN_IP=$(ip route show default | awk '{print $3}')

# 设置 HTTP/HTTPS 代理
export http_proxy=http://$WIN_IP:7890
export https_proxy=http://$WIN_IP:7890

# 设置 Git 代理
git config --global http.proxy http://$WIN_IP:7890
```

**常见代理软件及端口**：

| 软件 | HTTP 代理端口 |
|------|-------------|
| Clash / Clash Verge / CFW | 7890 |
| V2RayN | 10809 |
| Shadowsocks | 1080 |
| Clash Meta | 7890 |

### 永久生效

添加到 `~/.bashrc` 或 `~/.zshrc`：

```bash
# 自动检测 Windows 主机 IP 并设置代理
export WIN_IP=$(ip route show default | awk '{print $3}')
export http_proxy=http://$WIN_IP:7890
export https_proxy=http://$WIN_IP:7890
```

### 网络互操作

- **Windows 访问 WSL**：通过 `localhost`（当 `localhostForwarding=true`）
- **WSL 访问 Windows**：通过 `$(ip route show default | awk '{print $3}')` 获取宿主 IP
- **防火墙**：Windows 防火墙可能阻止 WSL 网络，需在防火墙中放行

## 常见问题排查

| 问题 | 解决方案 |
|------|----------|
| WSL 无法启动 | 管理员运行 `wsl --shutdown` 后重试 |
| 内存占用过高 | 编辑 `.wslconfig` 限制内存并重启 |
| 无法访问 localhost | 确保 `localhostForwarding=true` |
| 磁盘空间不足 | 使用 `wsl --export` 迁移，或启用 `sparseVhd` |
| 网络不通 | 检查 Windows 防火墙，尝试 `wsl --shutdown` 后重启 |
| DNS 不解析 | 编辑 `/etc/resolv.conf` 或 `/etc/wsl.conf` 禁用自动生成 |
| 跨系统文件性能差 | 将项目文件放在 WSL 内部（`/home/`）而非 `/mnt/c/` |

## Scoop 包管理器（Windows）

### 安装

```powershell
# 远程执行安装脚本
iwr -useb get.scoop.sh | iex

# 更新 Scoop 自身
scoop update

# 配置代理
scoop config proxy 127.0.0.1:10809
```

### 常用命令

| 命令 | 说明 |
|------|------|
| `scoop search <keyword>` | 搜索软件 |
| `scoop install <app>` | 安装软件 |
| `scoop uninstall <app>` | 卸载软件 |
| `scoop update` | 更新 bucket 和 Scoop 自身 |
| `scoop update *` | 更新所有已安装软件 |
| `scoop hold <app>` | 锁定软件阻止更新 |
| `scoop unhold <app>` | 解锁 |
| `scoop info <app>` | 查询软件详细信息 |
| `scoop home <app>` | 打开软件官网 |
| `scoop list` | 列出已安装软件 |
| `scoop status` | 检查可更新软件 |
| `scoop cleanup` | 清理旧版本 |
| `scoop bucket add <name>` | 添加仓库 |

### 常用 Bucket

| Bucket | 内容 | 添加命令 |
|--------|------|----------|
| `main` | 默认 CLI 应用 | 内置 |
| `extras` | GUI 应用 | `scoop bucket add extras` |
| `versions` | 软件替代/预发布版本 | `scoop bucket add versions` |
| `nerd-fonts` | Nerd Fonts 字体 | `scoop bucket add nerd-fonts` |
| `java` | JDK/JRE | `scoop bucket add java` |
| `games` | 开源游戏工具 | `scoop bucket add games` |
| `nonportable` | 非便携应用（需 UAC） | `scoop bucket add nonportable` |

### 通过 Scoop 安装常用工具

```powershell
# 开发工具
scoop install git
scoop install neovim
scoop install lazygit
scoop install fzf
scoop install ripgrep
scoop install fd

# 语言运行时
scoop install python
scoop install nodejs

# 终端
scoop bucket add extras
scoop install extras/alacritty
scoop install extras/windows-terminal
```

## Windows 常用软件

### 开发工具

- **VSCode** — 主力编辑器
- **Git** — 版本控制
- **MSYS2** — C++ 编译环境（参考 [VSCode C++ 教程](https://code.visualstudio.com/docs/cpp/config-mingw)）

### 终端与字体

- **Windows Terminal** — 微软官方多标签终端
- **PowerShell 7** — 跨平台 PowerShell
- **Alacritty** — GPU 加速 Rust 终端
- **FiraCode Nerd Font** — 带 ligature 的等宽字体，[Nerd Fonts 下载](https://www.nerdfonts.com/font-downloads)

### 效率工具

- **uTools** — 搜索/启动/效率工具箱
- **7-Zip** / **Bandizip** — 压缩解压
- **Fences** — 桌面图标整理

### 虚拟机与容器

- **WSL2** — Windows 内置 Linux 子系统（Ubuntu / Arch）
- **Docker Desktop** — 容器运行时（可基于 WSL2 后端）

## NVM Node 版本管理

### Windows 安装

推荐 [nvm-windows](https://github.com/coreybutler/nvm-windows)：

```powershell
# 管理员运行安装后验证
nvm version
```

### Linux/macOS 安装

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
source ~/.bashrc
nvm --version
```

### 常用命令

| 命令 | 说明 |
|------|------|
| `nvm install <version>` | 安装指定版本 |
| `nvm install node` | 安装最新版 |
| `nvm install --lts` | 安装最新 LTS |
| `nvm use <version>` | 切换到指定版本 |
| `nvm ls` | 列出已安装版本 |
| `nvm ls-remote` | 列出可安装版本（仅 Linux/macOS） |
| `nvm alias default <version>` | 设置默认版本 |
| `nvm uninstall <version>` | 卸载指定版本 |
| `nvm current` | 显示当前版本 |

```bash
# 常用操作流水线
nvm install 20        # 安装 Node 20
nvm use 20            # 切换
nvm alias default 20  # 设为默认
```

## AI 开发环境搭建

### 前置要求

- Node.js >= 18（通过 NVM 管理）
- 网络代理（国内网络环境）

### 工具清单

| 工具 | 用途 |
|------|------|
| [CCSwitch](https://github.com/farion1231/cc-switch) | 跨平台 AI API 切换/代理 |
| [Claude Code](https://claude.ai/code) | Anthropic AI 编程助手 |
| [CCSwitch CLI](https://github.com/SaladDay/cc-switch-cli) | CCSwitch 命令行版本 |
| [MiniMax](https://platform.minimax.com) | 国产 AI 模型 API 提供商 |

### 安装 Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code

# 验证
claude --version
```

### 配置 CCSwitch

**方式一：桌面应用**

1. 下载安装 CCSwitch
2. 在设置中添加 Claude Code 栏位
3. 开启代理功能
4. 配置 AI API Key（推荐 MiniMax）

**方式二：CLI**

```bash
curl -fsSL https://github.com/SaladDay/cc-switch-cli/releases/latest/download/install.sh | bash
```

验证：

```bash
ccswitch status
```

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| 安装慢 | 使用淘宝镜像 `npm config set registry https://registry.npmmirror.com` |
| 代理不生效 | 检查 CCSwitch 代理开关和系统代理设置 |
| API Key 无效 | 确认 MiniMax 账户余额充足，检查 API Key 是否正确 |
| npm 安装权限问题 | Linux 上不要用 sudo 装全局包，已用 NVM 则不需要 |

## Linux 补充安装

### dotnet SDK

```shell
# Ubuntu 22.04+
sudo apt-get install -y dotnet-sdk-8.0

# 验证
dotnet --version
```

### 一键环境初始化脚本

适用于新 WSL/Ubuntu 环境：

```shell
#!/bin/bash
# 设置代理（按需修改端口）
export WIN_IP=$(ip route show default | awk '{print $3}')
export http_proxy=http://$WIN_IP:7897
export https_proxy=http://$WIN_IP:7897

# 系统更新
sudo apt update
sudo apt upgrade -y

# 基础工具
sudo apt install -y curl git

# Git 全局配置
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# NVM + Node
sudo apt install -y libatomic1
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
source ~/.bashrc
nvm install nodejs

# Claude Code
npm install -g @anthropic-ai/claude-code
```

### 远程开发（VSCode Remote-SSH）

```shell
# 安装 SSH 服务端和辅助工具
sudo apt install openssh-server sshpass wget

# 启动 SSH 服务
sudo service ssh start
```

在 Windows VSCode 中安装 **Remote - SSH** 扩展，通过 `ssh username@wsl_ip` 连接。

## 相关资源

- [WSL 官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/)
- [WSL2 GitHub](https://github.com/microsoft/WSL2)
- [NVM 官方](https://github.com/nvm-sh/nvm)
- [NVM for Windows](https://github.com/coreybutler/nvm-windows)
- [Scoop 官方](https://scoop.sh/)
- [CCSwitch](https://github.com/farion1231/cc-switch)
- [Claude Code 文档](https://docs.anthropic.com/en/docs/claude-code)
