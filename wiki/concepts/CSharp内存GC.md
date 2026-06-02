---
title: "CSharp内存GC"
tags:
  - csharp
  - gc
  - memory
  - boxing
  - idisposable
created: 2026-06-02
---

# CSharp内存GC

C# 内存管理与垃圾回收的深度解析，涵盖值类型/引用类型分配、GC 分代回收、装箱拆箱优化、资源释放模式。

## 核心概念

- **分代 GC**：Gen0（短期）→ Gen1（幸存）→ Gen2（长期），越短回收越频繁
- **装箱**：值类型 → object/接口，堆分配 + 内存复制；三大避免策略：重载基类方法、泛型约束、统一接口
- **闭包 GC**：Lambda 捕获外部变量 → 编译器生成匿名类 → 每次调用可能产生堆分配
- **Dispose 模式**：`IDisposable` + `Dispose(bool)` 区分托管/非托管资源；`GC.SuppressFinalize` 避免重复清理

## List 中的 struct vs class

在 `List<T>` 中存储百万级元素时：
- **class**：100 万个独立堆对象需要 GC 逐个扫描、移动、更新引用
- **struct**：仅 1 个堆对象（内部数组），数据连续、CPU 缓存友好

## 与已有页面关联

- [[CSharp值类型性能]] — struct 在集合中的性能优势深入分析
- [[CSharp异步模型]] — ValueTask 通过值类型减少异步方法的堆分配
- [[CSharp集合框架]] — List 扩容导致的 GC 与 struct 优化实践
- [[CSharp委托特性]] — Lambda 闭包的 GC 影响与优化策略
- [[concepts/Unity性能优化|Unity 性能优化]] — BoehmGC 机制与 C# 分代 GC 的对比及优化策略

## 来源

- [[csharp-memory-gc-摘要]]
