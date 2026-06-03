---
title: "EmmyLua 环境安装与配置"
type: source-summary
updated: 2026-06-02
source: "raw/cs/languages/emmyLua-environment-setup.md"
tags: [lua, emmylua, luals, ide, rider, vscode, debug]
---

# EmmyLua 环境安装与配置

## 来源信息

- **原始文件**：`raw/cs/languages/emmyLua-environment-setup.md`
- **类型**：工具安装配置指南

## 要点

- EmmyLua 是 Lua 静态类型注解系统与语言服务器，现由 `emmylua-analyzer-rust` 驱动，生态包含 JetBrains 插件和 VS Code 插件
- **LuaLS v3.0+** 与原始 EmmyLua（IntelliJ 插件）注解风格不完全兼容，推荐以 Rust 分析器生态为准
- `.emmyrc.json` 是项目级配置文件，Rider 和 VS Code 共用，放置在项目根目录
- 关键配置段：`workspace`（工作区路径与 library）、`runtime`（Lua 版本与 require 模式）、`diagnostics`（诊断规则与全局变量白名单）、`completion`（补全行为）、`hint`（内联提示）
- Debug 推荐使用 VS Code Attach 模式，Rider 内置 Debugger 存在断点命中问题
- XLua 项目可用 SnippetGenerator 反射生成 Unity API 类型片段

## 关联页面

- [[entities/EmmyLua|EmmyLua]] — EmmyLua/LuaLS 工具实体
- [[concepts/EmmyLua注解系统|EmmyLua 注解系统]] — 注解驱动开发概念
