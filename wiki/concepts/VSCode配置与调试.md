---
title: "VSCode 配置与调试"
type: concept
updated: 2026-06-02
tags: [vscode, ide, debug, editor, configuration]
aliases: [VSCode配置, VSCode调试, tasks.json, launch.json]
---

# VSCode 配置与调试

VSCode 编辑器的配置、任务与调试体系。涵盖 tasks.json 自动化、launch.json 多目标调试、Vim/Neovim 插件集成、LeetCode 刷题环境。

## 核心体系

### 多配置管理

Profile 机制独立管理插件/设置/快捷键，支持团队共享。推荐基础插件：GitLens、IntelliCode、Todo Tree、WSL。

详见 [[sources/vscode-config-debug-摘要|VSCode 来源摘要]]。

### Tasks 任务系统

`tasks.json` 驱动自动化：

```json
{ "label": "Build", "type": "shell", "command": "cmake --build .",
  "group": "build", "dependsOn": ["PreBuild"] }
```

- **Shell vs Process**：命令解释 vs 直接执行
- **平台覆盖**：`windows`/`linux`/`osx` 属性
- **复合任务**：`dependsOn` + `dependsOrder: sequence`
- **变量替换**：`${workspaceFolder}` / `${file}` / `${config:name}`
- **全局任务**：`Tasks: Open User Tasks`

### Launch 调试配置

`launch.json` 驱动调试会话：

- **必选**：`type`（调试器）、`request`（launch/attach）、`name`
- **常用**：`program`、`args`、`env`、`cwd`、`preLaunchTask`、`postDebugTask`
- **多目标**：`compounds` 同时启动 Server + Client
- **serverReadyAction**：自动捕获端口打开浏览器
- **断点类型**：行内（Shift+F9）、函数、数据断点

### Vim/Neovim 插件

- **EasyMotion** — 屏幕任意位置跳转
- **vim-surround** — 包围字符快速修改
- **vim-indent-object** — 缩进级别选择（`ii`/`ai`/`aI`）
- **vim-sneak** — 双字符跳转（`s{cc}`/`S{cc}`）
- **CamelCaseMotion** — 驼峰/蛇形命名内移动
- **Neovim + LazyVim** — Scoop 安装、NerdFont、treesitter、lazygit 整合

### LeetCode 插件

- **区域测试**：`@lcpr case=start/end` 注释块
- **自定义参数**：`funName` + `paramTypes`（number/string/ListNode/TreeNode 等）

### 常用快捷键

| 操作 | 快捷键 |
|------|--------|
| 快速打开文件 | `Ctrl+P` |
| 符号跳转（文件） | `Ctrl+Shift+O` |
| 符号跳转（工作区） | `Ctrl+T` |
| 多光标 | `Alt+Click` |
| 列选择 | `Shift+Alt+拖拽` |
| 重命名符号 | `F2` |
| 问题面板 | `Ctrl+Shift+M` |

## 关联页面

- [[sources/vscode-config-debug-摘要|VSCode 来源摘要]]
- [[concepts/EmmyLua注解系统|EmmyLua 注解]] — Lua 开发
