---
title: "C# Struct 装箱优化与 GC 压力控制 — 摘要"
type: source-summary
updated: 2026-05-10
tags: [csharp, struct, boxing, gc, optimization, memory]
source: "raw/cs/languages/csharp-struct-boxing-gc.md"
---

# C# Struct 装箱优化与 GC 压力控制 — 摘要

来源：`raw/cs/languages/csharp-struct-boxing-gc.md`

## 概述

深入 C# 值类型装箱的三种触发场景，以及 `readonly struct`、`ref struct`、`IEquatable<T>`、`stackalloc`、`Span<T>`、InlineArray、CollectionsMarshal 等现代零分配优化策略。

## 要点

- **装箱本质**：值类型→`object`/接口引用时 CLR 在堆上分配新对象并复制值，产生 Gen0 GC 压力和 CPU 开销
- **三种装箱场景**：未重载的虚方法（`GetType()`）、作为 `object`/非泛型接口参数传递、Dictionary Key 未实现 `IEquatable<T>` 导致 `object.Equals()` 回退
- **泛型是第一工具**：CLR 具现化模型为每个值类型生成专用机器码，接口调用变直接调用
- **`readonly struct`**：声明不可变意图，消除 `in` 参数/`readonly ref` 访问时的防御性拷贝
- **`ref struct`**：最强装箱保证 — 编译器强制不可装箱、不可作为 class 字段、不可被闭包捕获（`Span<T>`、`ReadOnlySpan<T>`）
- **`IEquatable<T>`**：防止 `Dictionary`/`HashSet` 查找时的隐藏装箱，或直接用 `readonly record struct`（自动生成）
- **`stackalloc` + `Span<T>`**：栈上临时缓冲区，零 GC
- **.NET 9+ JIT 逃逸分析**：保守的自动栈分配优化，不可依赖
- **大 struct（≥64 byte）优先用 `class`**：复制成本超过分配节省

## 关联页面

- [[concepts/CSharp值类型性能|C# 值类型与 GC]] — 概念综合页
