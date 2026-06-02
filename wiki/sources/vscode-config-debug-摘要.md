---
title: "VSCode 配置与调试 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/tools/ide/vscode-config-debug.md"
tags: [vscode, ide, debug, editor, configuration]
---

# VSCode 配置与调试完全指南

## 来源

`raw/tools/ide/vscode-config-debug.md` — VSCode 配置与调试完整指南：多配置文件、tasks.json 任务系统、launch.json 调试配置、Vim/Neovim 插件、LeetCode 插件、C++ 调试技巧、快捷键

## 要点

1. **多配置文件（Profile）** — 独立管理插件/设置/快捷键，推荐基础插件集（GitLens、IntelliCode、Todo Tree 等）
2. **Tasks 任务系统** — `type`（shell/process）、`command`、`args`、平台特定属性（`windows/linux/osx`）、`dependsOn` 组合任务、`dependsOrder` 顺序控制、变量替换、background watching tasks
3. **Launch 调试配置** — `type`/`request`/`name` 必选属性、`preLaunchTask`/`postDebugTask`、`serverReadyAction` 自动打开 URL、多目标调试（compounds）、断点类型（行内/函数/数据）、重定向输入输出
4. **Vim 插件生态** — EasyMotion（快速跳转）、vim-surround（包围字符）、vim-commentary（注释）、vim-indent-object（缩进选择）、vim-sneak（双字符跳转）、CamelCaseMotion（驼峰移动）
5. **Neovim 集成** — Scoop 安装、LazyVim 配置、依赖（NerdFont/lazygit/treesitter/fzf/rg/fd）
6. **LeetCode 插件** — 区域测试（`@lcpr case=start/end`）、自定义参数类型（funName/paramTypes）
7. **常用技巧** — 命令行参数、多光标编辑、符号导航、Snippets 自定义、VSCode 作为 Git diff/merge 工具

## 关联 Wiki 页面

- [[concepts/VSCode配置与调试|VSCode 配置与调试]] — 概念页
