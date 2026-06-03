---
title: "EmmyLua 注解系统"
type: concept
updated: 2026-06-02
tags: [lua, emmylua, annotations, type-system, static-typing, luacats]
---

# EmmyLua 注解系统

## 概念

EmmyLua 注解系统（LuaCATS — Lua Comment And Type System）是一种通过**特殊注释**为动态类型 Lua 添加静态类型标注的方法。注解不改变 Lua 运行时行为，仅被语言服务器解析用于 IDE 功能。

核心理念：**注释即类型声明**——在保持 Lua 零运行时开销的前提下，获得静态类型语言的工具链体验。

## 设计动机

Lua 作为动态类型语言，缺乏编译期类型信息，导致 IDE 难以提供准确的代码补全、重构和错误检测。EmmyLua 注解通过约定格式的注释桥接此缺口：

- **开发者**写入类型注解作为文档和约束
- **语言服务器**解析注解模拟类型检查
- **IDE** 基于推断/声明的类型提供智能提示、跳转、查找引用

## 注解分类

| 分类 | 注解 | 核心用途 |
|------|------|----------|
| 类型系统 | `@class`, `@field`, `@alias`, `@type`, `@enum`, `@generic` | 定义类、字段、别名、枚举、泛型 |
| 函数签名 | `@param`, `@return`, `@overload`, `@vararg` | 声明参数和返回值类型 |
| 异步与约束 | `@async`, `@nodiscard` | 标记异步函数、禁止忽略返回值 |
| 类型操作 | `@cast`, `@as` | 运行时类型转换声明 |
| 模块系统 | `@module`, `@meta`, `@package` | 模拟模块导入、标记定义文件 |
| 运算符 | `@operator` | 声明算符元方法类型签名 |
| 代码质量 | `@deprecated`, `@diagnostic` | 弃用标记、诊断控制 |
| 文档引用 | `@see`, `@version`, `@source` | 交叉引用与元数据 |
| IDE 辅助 | `--region`/`--endregion`, `--- @language` | 代码折叠、内嵌语言 |

## 项目配置

注解系统通过项目根目录的 `.emmyrc.json` 配置：

- **workspace** — 工作区路径、类型库目录（library）、排除目录
- **runtime** — Lua 版本、require 解析模式、非标准符号
- **diagnostics** — 启用的诊断规则、全局变量白名单
- **completion / hint** — 补全行为、内联类型提示

## 生态考量

原始 EmmyLua（IntelliJ 插件）与社区分支 LuaLS v3.0+ 之间存在注解兼容性差异。选择新项目时优先以 LuaLS/LuaCATS 规范为准，此规范是 `emmylua-analyzer-rust` 的原生标注格式。

## 关联页面

- [[entities/EmmyLua|EmmyLua]] — 工具实体与生态概述
- [[sources/emmyLua-annotations-reference-摘要|注解完全参考]] — 每种注解的语法、示例和最佳实践
- [[sources/emmyLua-environment-setup-摘要|环境安装与配置]] — 搭建 IDE 环境
