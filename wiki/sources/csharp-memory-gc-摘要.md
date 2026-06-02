---
title: "CSharp内存GC-摘要"
source: "raw/cs/languages/csharp-memory-gc.md"
type: source-summary
created: 2026-06-02
---

# CSharp内存GC-摘要

> 源文件: [[CSharp内存GC]]

## 核心内容

系统讲解 C# 内存管理：托管堆与栈的分配机制、GC 分代回收原理（Gen0/1/2）、装箱/拆箱的触发场景与三大避免策略、struct vs class 在 List 中的性能差异、Lambda 闭包 GC 问题（编译器匿名类生成）、WeakReference 弱引用模式、IDisposable Dispose 模式（含托管/非托管资源区分）、GC 优化实践清单。

## 关键要点

1. **装箱三步**：堆分配 → 值复制 → 返回引用；避免方法：重载基类方法、泛型约束、统一接口
2. **List 中 struct 优势**：百万级元素仅 1 个堆对象（内部数组），class 有百万个独立堆对象
3. **闭包原理**：Lambda 捕获外部变量 → 编译器生成匿名类 → 每次调用可能产生新堆对象
4. **Dispose 模式**：`Dispose(bool)` 区分托管/非托管资源；`GC.SuppressFinalize` 避免重复清理
5. **WeakReference**：允许 GC 回收缓存对象，用 `TryGetTarget` 检测存活，适合非强制性引用

## 相关页面

- [[CSharp值类型性能]] — 已有 wiki 页面
- [[CSharp异步模型]]
- [[CSharp集合框架]]
