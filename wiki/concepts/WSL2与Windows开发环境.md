---
title: "WSL2 与 Windows 开发环境"
type: concept
updated: 2026-06-02
tags: [wsl, wsl2, windows, scoop, nvm, node, ai, dev-env]
aliases: [WSL2配置, Windows开发环境, WSL教程, 开发环境搭建]
---

# WSL2 与 Windows 开发环境

Windows + WSL2 混合开发环境的完整搭建方案：Linux 子系统安装/优化/迁移、包管理器（Scoop）、Node 版本管理（NVM）、AI 编程环境（Claude Code）。

## 核心体系

### WSL2 安装与迁移

```powershell
wsl --install                           # 一键安装
wsl --list --online                     # 查看可用发行版
wsl --export Ubuntu D:\ubuntu.tar       # 迁移：导出
wsl --unregister Ubuntu                 # 注销原位置
wsl --import Ubuntu E:\WSL\Ubuntu D:\ubuntu.tar  # 导入新位置
```

详见 [[sources/wsl-windows-dev-setup-摘要|WSL2 来源摘要]]。

### 性能调优

`%USERPROFILE%\.wslconfig`：

- `memory=8GB` — 限制最大内存（总内存 50%）
- `processors=4` — CPU 核心数
- `swap=2GB` — 交换空间
- `localhostForwarding=true` — 允许 Windows 访问 WSL 服务
- `autoMemoryReclaim=gradual` — 自动回收空闲内存

### 代理配置

WSL2 通过网关 IP 访问 Windows 代理：

```bash
export WIN_IP=$(ip route show default | awk '{print $3}')
export http_proxy=http://$WIN_IP:7890
# Clash: 7890, V2RayN: 10809, Shadowsocks: 1080
```

### Scoop — Windows 包管理

```powershell
iwr -useb get.scoop.sh | iex       # 安装
scoop install git neovim lazygit   # 开发工具
scoop bucket add extras            # GUI 应用
```

核心 bucket：`main`（CLI）、`extras`（GUI）、`versions`（预发布）、`nerd-fonts`（字体）。

### NVM — Node 版本管理

```bash
# Linux
nvm install 20 && nvm use 20 && nvm alias default 20

# Windows
nvm install 20.0.0 && nvm use 20.0.0
```

### AI 开发环境

```bash
npm install -g @anthropic-ai/claude-code   # Claude Code CLI
curl ... | bash                             # CCSwitch CLI（API 代理）
```

工具链：Claude Code + CCSwitch（API 切换/代理）+ MiniMax（国产模型 API）。

### 常见问题

| 问题 | 解决 |
|------|------|
| WSL 不启动 | `wsl --shutdown` 管理员运行 |
| 内存泄漏 | `.wslconfig` 限制 + `autoMemoryReclaim` |
| DNS 不解析 | 编辑 `/etc/wsl.conf` 禁用自动生成 |
| 跨系统文件慢 | 项目放 `/home/` 而非 `/mnt/c/` |

## 关联页面

- [[sources/wsl-windows-dev-setup-摘要|WSL2 来源摘要]]
- [[concepts/Linux Shell环境|Linux Shell 环境]]
- [[concepts/Git操作完全指南|Git 操作指南]]
