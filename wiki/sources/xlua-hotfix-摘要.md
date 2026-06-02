---
title: "XLua 互调与热补丁 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [xlua, unity, lua, csharp, hotfix, il-injection]
source: "raw/cs/languages/xlua-hotfix.md"
---

# XLua 互调与热补丁 — 摘要

来源：`raw/cs/languages/xlua-hotfix.md`

## 概述

XLua 框架核心机制：C# ↔ Lua 双向互调的完整映射方式（class/interface/delegate/LuaTable）、性能陷阱与优化策略、基于 IL 注入的热补丁实现原理（代码生成 → DelegateBridge → Mono.Cecil 注入）。

## 要点

- **C# 调用 Lua Table**：`class`（值拷贝，只读）、`interface`（引用拷贝，实时同步）、`Dictionary`/`List`（简单场景）、`LuaTable`（不推荐，慢 ~10x）
- **C# 调用 Lua Function**：Delegate（推荐，类型安全）+ `[CSharpCallLua]`；`LuaFunction`（不推荐）；多返回值通过 `out`/`ref` 从左往右映射
- **Lua 调用 C#**：通过 `CS.` 命名空间访问，`ref`/`out` 以多返回值体现，泛型容器通过构造函数形式获取类型实例
- **字典访问特殊性**：不能用 `dic["key"]`，必须用 `get_Item`/`set_Item`/`TryGetValue`
- **性能优化**：持有常用对象引用避免反复创建 userdata；用静态方法减少 object 查找；`Vector3`/`Quaternion` 传参开销极大，用三个 float 替代
- **`[GCOptimize]`**：值类型传递不产生 GC alloc，避免装箱
- **`[LuaCallCSharp]`** / **`[CSharpCallLua]`**：生成适配代码避免反射，所有互调类型必须标注
- **内存泄漏**：Lua 持有 C# 对象 → ObjectTranslator dictionary 续命 → 即使 Destroy 仍残留
- **热补丁流程**：`HOTFIX_ENABLE` 宏 → Generate Code（创建 DelegateBridge）→ Hotfix Inject（Mono.Cecil IL 注入）→ `xlua.hotfix()` 运行时替换
- **IL 注入核心**：每个 `[Hotfix]` 方法注入 `DelegateBridge` 静态变量，非空时执行 Lua 逻辑，为空时回退原始 C# 逻辑

## 关联页面

- [[concepts/XLua热补丁|XLua 热补丁]] — 概念综合页
- [[concepts/Lua核心特性|Lua 核心特性]]
