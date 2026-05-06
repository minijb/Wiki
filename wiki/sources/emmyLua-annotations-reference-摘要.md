---
title: "EmmyLua 注解完全参考"
type: source-summary
updated: 2026-05-07
source: "raw/cs/languages/emmyLua-annotations-reference.md"
tags: [lua, emmylua, annotations, type-system, luacats]
---

# EmmyLua 注解完全参考

## 来源信息

- **原始文件**：`raw/cs/languages/emmyLua-annotations-reference.md`
- **类型**：技术参考文档

## 要点

- 涵盖约 25 种 EmmyLua/LuaCATS 注解，按功能分为 9 组：类型系统、函数签名、异步与约束、类型操作、模块系统、运算符、代码质量、文档引用、IDE 辅助
- **类型系统**：`@class`（含精确类/部分类）、`@field`（访问控制 public/protected/private）、`@alias`（类型别名与字面量枚举）、`@type`（变量类型声明）、`@enum`（值枚举/键枚举）、`@generic`（泛型函数与泛型类，支持约束和捕获模式）
- **函数签名**：`@param`（含可选参数和可变参数）、`@return`（多返回值与可空返回）、`@overload`（多签名重载）、`@vararg`
- **类型操作**：`@cast`（添加/移除/覆盖类型）、`@as`（内联类型断言）
- **模块与元数据**：`@module`（模拟 require）、`@meta`（纯类型定义文件）、`@package`（文件级私有）
- **运算符**：`@operator` 支持 add/sub/mul/div/unm/concat/call 等 14 种算符类型推断
- **代码质量**：`@deprecated`（弃用标记）、`@diagnostic`（行级/文件级诊断开关）
- **IDE 辅助**：`--region`/`--endregion` 和 `--{{{`/`--}}}`（代码折叠）、`--- @language`（内嵌语言语法高亮）

## 关联页面

- [[entities/EmmyLua|EmmyLua]] — EmmyLua/LuaLS 工具实体
- [[concepts/EmmyLua注解系统|EmmyLua 注解系统]] — 注解驱动开发概念
