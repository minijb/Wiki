---
title: "VSCode 配置与调试完全指南"
date: 2026-06-02
tags: [vscode, ide, debug, editor, configuration]
type: tool
aliases: [VSCode配置, VSCode调试, VSCode教程, tasks.json, launch.json]
description: VSCode 配置与调试完整指南：多配置文件管理、tasks.json 任务系统、launch.json 调试配置、变量替换、平台特定配置、Vim/Neovim 插件、LeetCode 插件、C++调试技巧、常用快捷键与代码片段
---

# VSCode 配置与调试完全指南

## 多配置文件

VSCode 支持多配置文件（Profile），左下角齿轮图标可创建。每个 Profile 独立管理插件、设置、快捷键。

### 基础配置常用插件

- Chinese (Simplified) — 中文语言包
- GitLens — Git 增强
- IntelliCode — AI 智能提示
- JSON — JSON 编辑支持
- Material Icon Theme — 图标主题
- One Dark Pro — 配色主题
- Path Intellisense — 路径补全
- Project Manager — 项目管理
- Test Explorer UI — 测试管理
- Todo Tree — TODO 高亮树
- WSL — WSL 集成

## Tasks 任务系统

### 基础任务模板

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Run tests",
      "type": "shell",
      "command": "./scripts/test.sh",
      "windows": {
        "command": ".\\scripts\\test.cmd"
      },
      "group": "test",
      "presentation": {
        "reveal": "always",
        "panel": "new"
      }
    }
  ]
}
```

### 关键属性

| 属性 | 说明 |
|------|------|
| `type` | `shell`（shell 命令）/ `process`（可执行进程） |
| `command` | 要执行的确切命令 |
| `windows` / `linux` / `osx` | 平台特定属性覆盖 |
| `group` | 任务分组；`test` 组可被 "Run Test Task" 统一运行 |
| `presentation` | 控制终端输出行为（`reveal`、`panel`） |
| `options` | 覆盖 `cwd`（工作目录）、`env`（环境变量）、`shell` |
| `runOptions` | 定义任务运行时机与方式 |
| `dependsOn` | 任务依赖，组合复合任务 |
| `dependsOrder` | `"sequence"` 按顺序执行 vs 默认并行 |
| `hide` | 在 Quick Pick 中隐藏（适合仅作为子任务） |

### Shell 命令引号

单条命令直接使用 `command`。含参数时使用 `args` 数组。

**引号规则**：命令内只能使用**单引号**，需自己控制引号时：

```json
{
  "label": "dir",
  "type": "shell",
  "command": "dir",
  "args": [
    { "value": "folder with spaces", "quoting": "escape" }
  ]
}
```

`quoting` 取值：`strong`（Linux/macOS 用 `'`，cmd 用 `"`）、`weak`（统一 `"`）。

### 复合任务（dependsOn）

```json
{
  "label": "Build",
  "dependsOn": ["Client Build", "Server Build"],
  "dependsOrder": "sequence"
}
```

### 平台特定属性

```json
{
  "label": "Run Node",
  "type": "process",
  "windows": { "command": "C:\\Program Files\\nodejs\\node.exe" },
  "linux": { "command": "/usr/bin/node" },
  "osx": { "command": "/usr/local/bin/node" }
}
```

### 变量替换

`command`、`args`、`options` 三属性支持变量替换，如 `${workspaceFolder}`、`${file}`、`${config:settingName}` 等。

参考：[VSCode Variables Reference](https://code.visualstudio.com/docs/editor/variables-reference)

### 全局任务

通过 `Tasks: Open User Tasks` 打开全局 `tasks.json`，定义跨项目的通用任务。

### 后台/监视任务

适用于 `tsc --watch`、`npm dev` 等持续运行的工具。配置参考 [VSCode Background Tasks](https://code.visualstudio.com/docs/editor/tasks#_background-watching-tasks)。

### Problem Matcher

定义问题匹配器，将工具输出解析为 VSCode 问题面板中的条目。支持单行和多行匹配。

### 快捷键绑定任务

```json
{
  "key": "ctrl+h",
  "command": "workbench.action.tasks.runTask",
  "args": "Run tests"
}
```

## Launch 调试配置

### 基础 C++ 调试配置

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "C/C++ Runner: Debug Session",
      "type": "cppdbg",
      "request": "launch",
      "args": [],
      "stopAtEntry": false,
      "externalConsole": true,
      "cwd": "d:/Project/leetcode",
      "program": "d:/Project/leetcode/build/Debug/outDebug",
      "MIMode": "gdb",
      "miDebuggerPath": "gdb",
      "setupCommands": [
        {
          "description": "Enable pretty-printing for gdb",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        }
      ]
    }
  ]
}
```

### 必选属性

| 属性 | 说明 |
|------|------|
| `type` | 调试器类型（每个 debug 扩展有唯一 type） |
| `request` | 请求类型：`launch`（启动）/ `attach`（附加） |
| `name` | 自定义配置名称 |

### 常用选项

| 属性 | 说明 |
|------|------|
| `program` | 调试的可执行文件路径 |
| `args` | 传递给程序的参数 |
| `env` | 环境变量（`null` 可取消变量） |
| `envFile` | dotenv 环境变量文件路径 |
| `cwd` | 工作目录 |
| `port` | attach 时的端口 |
| `stopOnEntry` | 启动时立即断点 |
| `console` | `internalConsole` / `integratedTerminal` / `externalTerminal` |
| `preLaunchTask` | 调试前执行的任务（对应 tasks.json 中 label） |
| `postDebugTask` | 调试结束后执行的任务 |

### 展示配置（presentation）

```json
"presentation": {
  "hidden": false,
  "group": "",
  "order": 1
}
```

### 自动打开 URL（serverReadyAction）

```json
{
  "serverReadyAction": {
    "pattern": "listening on port ([0-9]+)",
    "uriFormat": "http://localhost:%s",
    "action": "openExternally"
  }
}
```

`pattern` 正则捕获终端输出，`%s` 被第一个捕获组替换。`action` 可选 `debugWithEdge` / `debugWithChrome` / `startDebugging`。

### 多目标调试（Multi-target）

同时启动多个调试进程：

```json
{
  "compounds": [
    {
      "name": "Server/Client",
      "configurations": ["Server", "Client"],
      "preLaunchTask": "${defaultBuildTask}",
      "stopAll": true
    }
  ]
}
```

### 平台特定属性

同 tasks，使用 `windows`、`linux`、`osx` 属性覆盖。

### 断点类型

- **行内断点**：`Shift+F9`
- **函数断点**：按函数名断点
- **数据断点**：变量值变化时中断

### 重定向输入/输出

```json
{
  "name": "launch program that reads a file from stdin",
  "type": "node",
  "request": "launch",
  "program": "program.js",
  "console": "integratedTerminal",
  "args": ["<", "in.txt"]
}
```

### 变量替换

同 tasks，支持 `${workspaceFolder}`、`${file}` 等变量。

## Vim 插件

### EasyMotion

快速跳转：

| 命令 | 说明 |
|------|------|
| `<leader><leader>s` | 查找字符 |
| `<leader><leader>f` | 向前查找字符 |
| `<leader><leader>F` | 向后查找字符 |
| `<leader><leader>w` | 单词起始 |
| `<leader><leader>b` | 单词起始（反向） |
| `<leader><leader>j` | 向下跳行 |
| `<leader><leader>k` | 向上跳行 |
| `<leader><leader>/` | 搜索跳转 |

### vim-surround

| 命令 | 说明 |
|------|------|
| `cs{old}{new}` | 替换包围字符 |
| `ys{motion}{char}` | 添加包围字符 |
| `ds{char}` | 删除包围字符 |
| `S{char}` | 在 visual 模式下包围选中内容 |

### vim-commentary

`gc` / `gC` — 注释/取消注释。

### vim-indent-object

基于缩进级别快速选中（常用于 if 块和 Python）：

| 命令 | 说明 |
|------|------|
| `<operator>ii` | 当前缩进级别 |
| `<operator>ai` | 当前缩进级别 + 上一行（Python if 风格） |
| `<operator>aI` | 当前缩进级别 + 上一行 + 下一行（C/C++/Java if 风格） |

### vim-sneak

双字符快速跳转：

| 命令 | 说明 |
|------|------|
| `s{char}{char}` | 向前跳转到首次出现位置 |
| `S{char}{char}` | 向后跳转 |
| `<operator>z{char}{char}` | 向前操作到首次出现位置 |
| `<operator>Z{char}{char}` | 向后操作到首次出现位置 |

使用 `;` / `,` 移动到下一个/上一个匹配。

### CamelCaseMotion

驼峰/蛇形命名内移动（替代标准 `w`）：

| 命令 | 说明 |
|------|------|
| `<leader>w` | 向前移动到下一驼峰/蛇形单词段起始 |
| `<leader>e` | 向前移动到下一驼峰/蛇形单词段末尾 |
| `<leader>b` | 向后移动到上一驼峰/蛇形单词段起始 |
| `<operator>i<leader>w` | 选择/修改当前驼峰/蛇形单词段 |

## Neovim 插件（VSCode 集成）

### 安装 Neovim

**Windows（Scoop）**：

```powershell
# 稳定版
scoop install neovim

# 开发版
scoop bucket add versions
scoop install neovim-nightly
```

**必备依赖**：
- Neovim >= 0.9.0（需 LuaJIT 构建）
- Git >= 2.19.0
- Nerdfont 字体（可选，用于图标显示）
- lazygit（可选）
- C 编译器（nvim-treesitter 需要）
- curl（blink.cmp 补全引擎需要）
- fzf、ripgrep、fd（可选，fzf-lua 插件需要）
- 支持 true color 和 undercurl 的终端：kitty、wezterm、alacritty、iterm2

### 安装 LazyVim

```powershell
# 备份现有配置
Move-Item $env:LOCALAPPDATA\nvim $env:LOCALAPPDATA\nvim.bak
Move-Item $env:LOCALAPPDATA\nvim-data $env:LOCALAPPDATA\nvim-data.bak

# 克隆 starter
git clone https://github.com/LazyVim/starter $env:LOCALAPPDATA\nvim

# 移除 .git 以便自行管理
Remove-Item $env:LOCALAPPDATA\nvim\.git -Recurse -Force

# 启动 Neovim
nvim
```

## C++ 调试技巧

### 查看泛型/STL 容器

在 VSCode C++ 调试中查看 STL 容器内容：

参考：
- [知乎：VSCode C++ Debug 泛型查看](https://zhuanlan.zhihu.com/p/351093679)
- [CSDN：VSCode 查看 STL 容器](https://blog.csdn.net/u014552102/article/details/126693028)

核心：配置 `setupCommands` 启用 pretty-printing，以及使用 Natvis 文件自定义显示。

## LeetCode 插件

### 区域测试

```cpp
// @lcpr case=start
// "PAYPALISHIRINGGGG"\n3\n
// @lcpr case=end
```

### 自定义参数类型

用于调试不同签名的函数：

```cpp
// @lcpr-div-debug-arg-start
// funName= alternateDigitSum
// paramTypes= ["number"]
// @lcpr-div-debug-arg-end
```

支持的 `paramTypes`：

| 值 | 类型 |
|------|------|
| `"number"` | 数字 |
| `"number[]"` | 数字数组 |
| `"number[][]"` | 数字二维数组 |
| `"string"` | 字符串 |
| `"string[]"` | 字符串数组 |
| `"string[][]"` | 字符串二维数组 |
| `"ListNode"` | 链表 |
| `"ListNode[]"` | 链表数组 |
| `"character"` | 字节 |
| `"character[]"` | 字节数组 |
| `"character[][]"` | 字节二维数组 |
| `"NestedInteger[]"` | 数组 |
| `"MountainArray"` | 数组 |
| `"TreeNode"` | 树节点 |

## 常用快捷键与技巧

### 命令行

```shell
code .                    # 打开当前目录
code -r .                 # 在最近窗口打开
code -n                   # 新建窗口
code --diff <file1> <file2>  # 打开差异编辑器
code --goto package.json:10:5   # 定位到文件行列
code --disable-extensions .      # 禁用所有扩展
code --locale=es           # 切换语言
```

### 导航

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+P` | 快速打开文件 |
| `Ctrl+Shift+E` | 显示资源管理器 |
| `Ctrl+T` | 工作区符号跳转 |
| `Ctrl+Shift+O` | 文件内符号跳转（加 `@:` 分组） |
| `Ctrl+Tab` | 浏览导航历史 |
| `Alt+Left/Right` | 前进/后退 |
| `Ctrl+Shift+M` | 问题面板（`F8`/`Shift+F8` 循环跳转） |

### 编辑

| 快捷键 | 功能 |
|--------|------|
| `Alt+Click` | 多光标 |
| `Shift+Alt+拖拽` | 列（矩形）选择 |
| `Alt+Up/Down` | 上下移动行 |
| `Shift+Alt+Left/Right` | 缩小/扩展选择 |
| `Ctrl+Shift+[` / `Ctrl+Shift+]` | 折叠/展开代码 |
| `Ctrl+K Ctrl+L` | 折叠/展开递归 |
| `F2` | 重命名符号 |

### Snippets 自定义

**文件** > **首选项** > **配置用户代码片段**，选择语言：

```json
"create component": {
    "prefix": "component",
    "body": [
        "class $1 extends React.Component {",
        "",
        "\trender() {",
        "\t\treturn ($2);",
        "\t}",
        "",
        "}"
    ]
}
```

### VSCode 作为 Git 工具

```shell
# 设置 VSCode 为默认合并工具
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# 设置 VSCode 为默认差异工具
git config --global diff.tool vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'
```

### C++ 设置

- 方法自动添加括号：`C_Cpp: Autocomplete Add Parentheses`

## 相关资源

- [VSCode Tasks 文档](https://code.visualstudio.com/docs/editor/tasks)
- [VSCode Tasks Appendix](https://code.visualstudio.com/docs/editor/tasks-appendix)
- [VSCode Debugging 文档](https://code.visualstudio.com/docs/editor/debugging)
- [VSCode Variables Reference](https://code.visualstudio.com/docs/editor/variables-reference)
- [VSCode Tips and Tricks](https://code.visualstudio.com/docs/getstarted/tips-and-tricks)
