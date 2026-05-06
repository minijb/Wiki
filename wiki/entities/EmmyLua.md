---
title: "EmmyLua"
type: entity
updated: 2026-05-07
aliases: [LuaLS, Lua Language Server, emmylua-analyzer-rust]
tags: [lua, emmylua, luals, language-server, tool, ide]
---

# EmmyLua

## 概述

EmmyLua 是 Lua 的静态类型注解系统与语言服务器（Language Server），为动态类型 Lua 提供 IDE 级智能提示、类型检查、代码导航和调试支持。

## 历史与生态

项目最初为 JetBrains IDE 的 Lua 插件（IntelliJ-EmmyLua），后被社区接手发展为更广泛的生态：

| 组件 | 说明 | 仓库 |
|------|------|------|
| emmylua-analyzer-rust | Rust 重写的分析器核心（当前主力） | [GitHub](https://github.com/EmmyLuaLs/emmylua-analyzer-rust) |
| Intellij-EmmyLua2 | JetBrains 插件（Rider/IDEA） | [GitHub](https://github.com/EmmyLua/Intellij-EmmyLua2) |
| VS Code 插件 | tangzx.emmylua | [Marketplace](https://marketplace.visualstudio.com/items?itemName=tangzx.emmylua) |
| LuaLS (Lua Language Server) | 社区分支的 LSP 实现 | [luals.github.io](https://luals.github.io/wiki/) |

## 核心机制

- **注解驱动** — 以 `---` 开头的特殊注释（如 `---@class`, `---@param`）为 Lua 代码添加静态类型信息
- **LuaCATS** — Lua Comment And Type System，注解系统的正式规范名称
- **`.emmyrc.json`** — 项目级配置文件，定义工作区、运行时、诊断规则等

## 兼容性

LuaLS v3.0+ 与原始 EmmyLua（IntelliJ 插件）的注解风格不完全兼容。目前推荐以 `emmylua-analyzer-rust` 和 LuaCATS 规范为准。

## 关联页面

- [[concepts/EmmyLua注解系统|EmmyLua 注解系统]] — 注解驱动 Lua 开发概念
- [[sources/emmyLua-annotations-reference-摘要|注解完全参考]] — 约 25 种注解的完整语法和示例
- [[sources/emmyLua-environment-setup-摘要|环境安装与配置]] — Rider/VS Code 搭建指南
