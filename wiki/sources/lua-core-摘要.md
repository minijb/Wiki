---
title: "Lua 核心特性 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [lua, table, closure, metatable, coroutine]
source: "raw/cs/languages/lua-core.md"
---

# Lua 核心特性 — 摘要

来源：`raw/cs/languages/lua-core.md`

## 概述

Lua 语言核心机制深度分析：Table 双段结构与 Rehash 算法、闭包与 UpValue 共享模型、元表系统与 OOP 模拟、协程调度机制、以及内存管理（三色标记 GC + 弱引用）。

## 要点

- **Table 双段结构**：数组段（连续整数键）+ 哈希段（其余键）。分配原则：数组至少一半利用率，后半部分至少一个元素
- **Rehash**：插入新键空间不足时触发，通过 `numusearray` / `numusehash` / `computesizes` 计算最优数组段大小后 `resize`
- **`#` 陷阱**：对含 nil 的 table 行为不可靠，安全获取长度需遍历 `pairs`
- **UpValue 共享**：多个闭包捕获同一外部变量时共享 UpValue，离开作用域后值提升到堆（closed）或回收
- **元表 OOP**：`__index` 指向自身实现原型链继承，`__newindex` 控制写入，`rawget`/`rawset` 绕过元方法
- **协程**：`create` + `resume` 返回 `(ok, value)`，`wrap` 返回函数更简洁但异常直接抛出
- **值传递 vs 引用传递**：`nil`/`boolean`/`number`/`string` 值拷贝，`table`/`function`/`userdata`/`thread` 引用传递
- **GC**：增量式三色标记-清除，支持弱引用表（`__mode = "k"/"v"/"kv"`）
- **`pairs` vs `ipairs`**：`ipairs` 从 1 开始遇 nil 停止，`pairs` 遍历所有键但顺序不确定，数值 for 性能最高

## 关联页面

- [[concepts/Lua核心特性|Lua 核心特性]] — 概念综合页
- [[concepts/XLua热补丁|XLua 热补丁]] — xLua 互调与热更新
