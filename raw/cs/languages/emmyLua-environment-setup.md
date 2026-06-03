---
title: "EmmyLua 环境安装与配置"
type: source
updated: 2026-06-02
tags: [lua, emmylua, luals, ide, rider, vscode, debug]
description: EmmyLua / LuaLS 语言服务器环境搭建：Rider 插件安装、VS Code 插件安装、.emmyrc.json 配置详解（workspace/runtime/diagnostics/completion/strict）、Debug 调试（Attach 模式）、参考资源
---

# EmmyLua 环境安装与配置

EmmyLua 是 Lua 的静态类型注解系统与语言服务器，为动态类型 Lua 提供 IDE 级智能提示、类型检查、代码导航和调试支持。项目原名 EmmyLua（IntelliJ 插件），后被社区接手发展为 **Lua Language Server (LuaLS)**，运行在 Rust 重写的分析器 `emmylua-analyzer-rust` 上。

核心组件：

| 组件 | 仓库 |
|------|------|
| Lua 分析器（Rust） | [EmmyLuaLs/emmylua-analyzer-rust](https://github.com/EmmyLuaLs/emmylua-analyzer-rust) |
| JetBrains 插件 | [EmmyLua/Intellij-EmmyLua2](https://github.com/EmmyLua/Intellij-EmmyLua2) |
| VS Code 插件 | [tangzx.emmylua](https://marketplace.visualstudio.com/items?itemName=tangzx.emmylua) |

> **兼容性提示：** LuaLS v3.0+ 与原始 EmmyLua（IntelliJ 插件）的注解风格不完全兼容。本文以 `emmylua-analyzer-rust` 生态为准。

## 1. Rider 插件安装

### 1.1 安装插件

1. 打开 Rider → **Settings** → **Plugins**
2. 搜索 **EmmyLua**（插件名 `Intellij-EmmyLua2`）
3. 点击 Install，重启 Rider

### 1.2 使用 SnippetGenerator（可选）

对于 XLua 项目，可使用 [SnippetGenerator](https://github.com/ak47007tiger/EmmyLuaXLuaSnippetGenerator) 通过反射生成 Unity API 的 EmmyLua 类型片段文件，放入 workspace library 后即可获得完整的 C# API 补全。

### 1.3 创建 `.emmyrc.json`

在项目根目录（与 `.sln` 同级）创建 `.emmyrc.json`，此为 EmmyLua 语言服务器的项目级配置文件。Rider 和 VS Code 共用此文件。

> **已知限制：** EmmyLua 只加载项目顶层第一个 `.emmyrc.json`，多项目 workspace 中其他 `.emmyrc.json` 会被忽略。

## 2. `.emmyrc.json` 配置详解

完整配置示例及字段说明：

### 2.1 workspace — 工作区

```json
"workspace": {
    "workspaceRoots": ["Assets/Script/Lua"],
    "library": [],
    "ignoreDir": [],
    "ignoreGlobs": [],
    "preloadFileSize": 0,
    "encoding": "utf-8",
    "moduleMap": [],
    "reindexDuration": 5000,
    "enableReindex": false
}
```

| 字段 | 说明 |
|------|------|
| `workspaceRoots` | 显式指定工作区根目录（Lua 脚本所在目录）。若不设置，分析器自动推断 |
| `library` | 类型存根/API 定义目录路径。放入第三方库或 C# API 的类型声明文件 |
| `ignoreDir` | 排除分析的目录。防止分析器索引不需要的版本文件夹 |
| `ignoreGlobs` | Glob 模式排除。如 `["**/generated/**"]` |
| `preloadFileSize` | 预加载文件大小阈值（KB）。超过的大文件不预加载，`0` 表示全部预加载 |
| `encoding` | 文件编码，默认 `utf-8` |
| `moduleMap` | 模块映射，将模块名映射到文件路径 |

### 2.2 runtime — 运行时

```json
"runtime": {
    "version": "LuaLatest",
    "requireLikeFunction": ["import", "load", "dofile"],
    "frameworkVersions": [],
    "extensions": [".lua", ".lua.txt", ".luau"],
    "requirePattern": ["?.lua", "?/init.lua", "lib/?.lua"],
    "nonstandardSymbol": ["continue"],
    "special": { "errorf": "error" }
}
```

| 字段 | 说明 |
|------|------|
| `version` | Lua 版本：`"Lua5.1"` / `"Lua5.2"` / `"Lua5.3"` / `"Lua5.4"` / `"LuaJIT"` / `"LuaLatest"` |
| `requireLikeFunction` | 视为 `require` 的函数名列表。如 XLua 的 `import` |
| `frameworkVersions` | 框架环境：`"love2d"`, `"openresty"`, `"nginx"` 等 |
| `extensions` | 视为 Lua 文件的扩展名列表 |
| `requirePattern` | require 路径解析模式。`?` 替换为模块名 |
| `nonstandardSymbol` | 非标准语法支持。如 `"continue"` 启用 continue 关键字 |
| `special.errorf` | 标记为 error 类函数（调用后该路径的类型收缩为 nil） |

### 2.3 diagnostics — 诊断

```json
"diagnostics": {
    "enable": true,
    "disable": [],
    "enables": [],
    "globals": [],
    "globalsRegex": [],
    "severity": {},
    "diagnosticInterval": 500
}
```

| 字段 | 说明 |
|------|------|
| `disable` | 全局禁用的诊断规则列表 |
| `enables` | 全局启用的额外诊断规则 |
| `globals` | 全局变量白名单。如 `["self", "UE", "CS"]` |
| `globalsRegex` | 全局变量正则白名单。如 `["^UE_.*"]` |
| `severity` | 单个诊断规则的严重级别覆盖 |
| `diagnosticInterval` | 诊断去抖间隔（ms），默认 500 |

常用全局白名单场景：
- XLua: `globals: ["CS", "UE", "xlua"]`
- 游戏脚本: `globals: ["self", "Game", "Player"]`

### 2.4 codeAction / codeLens / completion / hint

```json
"codeAction": { "insertSpace": false },
"codeLens": { "enable": true },
"completion": {
    "enable": true,
    "autoRequire": true,
    "autoRequireFunction": "require",
    "autoRequireNamingConvention": "keep",
    "autoRequireSeparator": ".",
    "callSnippet": false,
    "postfix": "@",
    "baseFunctionIncludesName": true
},
"hint": {
    "enable": true,
    "paramHint": true,
    "indexHint": true,
    "localHint": true,
    "overrideHint": true,
    "metaCallHint": true
}
```

| completion 字段 | 说明 |
|-----------------|------|
| `autoRequire` | 补全时自动添加 require 语句 |
| `autoRequireFunction` | auto require 使用的函数名 |
| `autoRequireNamingConvention` | 自动命名约定：`"keep"` / `"camelCase"` / `"snake_case"` |
| `autoRequireSeparator` | 模块路径分隔符 |
| `callSnippet` | 函数补全后是否作为代码片段（可 Tab 跳转参数） |
| `postfix` | 后缀补全触发符。`"@"` 表示 `local x = expr@` → `local x = type(expr)` |

### 2.5 strict — 严格模式

```json
"strict": {
    "requirePath": false,
    "typeCall": false,
    "arrayIndex": true,
    "metaOverrideFileDefine": true,
    "docBaseConstMatchBaseType": true
}
```

| 字段 | 说明 |
|------|------|
| `requirePath` | 严格检查 require 路径是否存在 |
| `arrayIndex` | 数组索引从 1 开始检查（Lua 约定） |
| `typeCall` | 严格检查 `()` 调用的类型 |
| `metaOverrideFileDefine` | meta 文件中可覆盖已有类型定义 |
| `docBaseConstMatchBaseType` | 注解中的基础类型需与实际基础类型匹配 |

### 2.6 其他配置

```json
"doc": { "syntax": "md" },
"references": {
    "enable": true,
    "fuzzySearch": true,
    "shortStringSearch": false
},
"semanticTokens": { "enable": true },
"signature": { "detailSignatureHelper": true }
```

- `doc.syntax` — 文档注释语法：`"md"`（Markdown）或 `"plain"`
- `references.fuzzySearch` — 模糊搜索查找引用
- `semanticTokens` — 语义高亮
- `signature.detailSignatureHelper` — 详细签名帮助

## 3. VS Code 插件安装

1. 打开 VS Code → **Extensions**（`Ctrl+Shift+X`）
2. 搜索 **EmmyLua**（发布者 `tangzx`）
3. 点击 Install
4. 在项目根目录创建 `.emmyrc.json`（与 Rider 共用同一配置）
5. 打开 Lua 文件即可自动激活语言服务器

VS Code 下 EmmyLua 使用 **lua-language-server** 作为后端，注解支持更接近 LuaLS/LuaCATS 规范。

## 4. Debug 调试

### 4.1 VS Code Attach 模式（推荐）

Rider 的 EmmyLua Debug 功能存在已知 bug，推荐使用 VS Code 的 Attach 模式：

1. 安装 VS Code 的 EmmyLua 插件（内含 Debugger）
2. 在 Unity 中启动游戏，确保 Lua 环境加载 EmmyLua Debug 模块
3. 在 VS Code 中创建 `.vscode/launch.json`：

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "emmylua_attach",
            "request": "attach",
            "name": "Attach to EmmyLua",
            "host": "localhost",
            "port": 9966
        }
    ]
}
```

4. 在 Lua 代码中设置断点
5. 按 F5 启动 Attach，连接到运行中的 Lua 进程

### 4.2 Rider Debug（有 bug）

Rider 内置的 EmmyLua Debugger 在部分版本中存在断点不命中、变量查看不完整等问题。若必须使用：
1. Run → Edit Configurations → Add → EmmyLua Debug
2. 配置 host/port
3. 对于 XLua 项目，确保 `luaEnv.DoString()` 传入了 `chunkName` 完整路径，否则断点无法解析

## 5. 参考资源

| 资源 | 链接 |
|------|------|
| EmmyLua Analyzer (Rust) | https://github.com/EmmyLuaLs/emmylua-analyzer-rust |
| IntelliJ-EmmyLua2 插件 | https://github.com/EmmyLua/Intellij-EmmyLua2 |
| VS Code 插件 | https://marketplace.visualstudio.com/items?itemName=tangzx.emmylua |
| SnippetGenerator (XLua) | https://github.com/ak47007tiger/EmmyLuaXLuaSnippetGenerator |
| LuaLS 注解文档 | https://luals.github.io/wiki/annotations/ |
| `.emmyrc.json` Schema | https://raw.githubusercontent.com/EmmyLuaLs/emmylua-analyzer-rust/refs/heads/main/crates/emmylua_code_analysis/resources/schema.json |
