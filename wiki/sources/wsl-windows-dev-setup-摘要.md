---
title: "WSL2 与 Windows 开发环境 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/tools/shell/wsl-windows-dev-setup.md"
tags: [wsl, wsl2, windows, scoop, nvm, node, ai]
---

# WSL2 与 Windows 开发环境搭建

## 来源

`raw/tools/shell/wsl-windows-dev-setup.md` — WSL2 与 Windows 开发环境完整搭建指南：WSL2 安装/迁移/配置/代理、Scoop 包管理器、Windows 工具清单、NVM 版本管理、AI 环境搭建

## 要点

1. **WSL2 安装与配置** — `wsl --install`、发行版管理（list/install/unregister）、WSL 命令速查表（管理/实例/导入导出/子系统内）
2. **磁盘迁移** — 导出（`--export`）→ 注销（`--unregister`）→ 导入（`--import`）到非系统盘
3. **性能调优** — `.wslconfig` 限制 memory/processors/swap、`autoMemoryReclaim`、`sparseVhd` 自动收缩
4. **代理配置** — WSL 网关 IP 获取（`ip route`）、永久生效（`.bashrc`/`.zshrc`）、常见代理端口映射
5. **Scoop 包管理器** — `iwr get.scoop.sh | iex` 安装、常用 bucket（main/extras/versions/nerd-fonts/java）、search/install/update/hold
6. **NVM Node 管理** — Windows（nvm-windows）+ Linux（nvm-sh）双平台安装、常用命令
7. **AI 环境** — Claude Code CLI（npm install）、CCSwitch 代理/API 切换、国产模型 API 集成
8. **一键初始化脚本** — curl/git/NVM/Node/Claude Code 自动化安装

## 关联 Wiki 页面

- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]] — 概念页
- [[concepts/Linux Shell环境|Linux Shell]] — Shell 指南
