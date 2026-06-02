---
title: "Linux Shell 环境"
type: concept
updated: 2026-06-02
tags: [linux, shell, bash, fish, zsh, terminal]
aliases: [Linux Shell, Shell环境, Bash教程, Fish配置]
---

# Linux Shell 环境

Linux 命令行环境完整配置：Bash 脚本编程、Fish/Zsh Shell 选择、终端多路复用、Neovim 开发环境、高效 CLI 工具链。

## 核心体系

### Bash 脚本编程

Linux 系统的默认脚本语言，从变量/数组/字符串到条件/循环/函数/信号的完整能力。

```bash
# 变量与替换
name=value
echo ${STRING//be/eat}         # 全局替换
echo ${STRING:1:3}             # 子串

# 文件测试
if [ -f "$file" ] && [ -r "$file" ]; then
    cat "$file"
fi
```

详见 [[sources/linux-shell-guide-摘要|Linux Shell 来源摘要]]。

### Shell 选择

| Shell | 特点 | 适用场景 |
|-------|------|----------|
| **Bash** | 标准、兼容性好、脚本首选 | 服务器、脚本 |
| **Fish** | 开箱即用、自然语法、自动建议 | 交互式终端 |
| **Zsh + OMZ** | 插件生态丰富、高度可定制 | 开发者日常 |

Fish 核心优势：无 `then`/`fi`/`do`/`done` 关键字、`()` 命令替代、Web 配置界面（`fish_config`）、Ctrl+R 历史搜索、自动补全建议。

Zsh + Oh My Zsh：一键安装、Antigen 插件管理、git/docker/extract 等内置插件。

### 终端多路复用

tmux 会话持久化、分屏操作（`Prefix %` 垂直 / `Prefix "` 水平）、断线重连。

### Neovim 开发环境

PPA 安装最新版 → LazyVim 配置 → 插件生态：

- **竞赛**：competitest.nvim、leetcode.nvim
- **笔记**：mkdnflow.nvim、neorg
- **大纲**：symbols-outline.nvim

### 高效工具链

```shell
lazygit   # 终端 Git GUI，可视化提交
fzf       # 模糊搜索，Ctrl+R/Ctrl+T/**
rg        # ripgrep，极速文本搜索
fd        # 现代 find 替代
```

### 文件系统基础

- **权限**：`chmod 755`（rwx）、`chown user:group`
- **目录**：`/usr`（软件）、`/etc`（配置）、`/var`（运行时数据）、`/tmp`（临时）

## 关联页面

- [[sources/linux-shell-guide-摘要|Linux Shell 来源摘要]]
- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]]
- [[concepts/C++核心语法|C++ 核心语法]] — C++ 开发环境
