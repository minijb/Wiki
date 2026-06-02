---
title: "CSharp集合框架"
tags:
  - csharp
  - collections
  - list
  - ienumerable
  - performance
created: 2026-06-02
---

# CSharp集合框架

C# 集合接口层次、List 内部实现、扩容策略与性能优化、Span/Memory 零分配操作。

## 核心概念

- **接口层次**：`IEnumerable<T>`（foreach）→ `ICollection<T>`（Add/Remove/Count）→ `IList<T>`（索引访问）→ `IReadOnlyList<T>`（只读语义）
- **List 扩容**：翻倍策略（0→4→8→16…）；小容量频繁扩容产生 GC，大容量复制导致内存峰值翻倍
- **优化策略**：预分配 `new List<T>(capacity)` 最简单有效；`TrimExcess()` 释放多余空间；`ArrayPool<T>` 临时数组租赁
- **Span\<T\>**：`CollectionsMarshal.AsSpan(list)` 零分配引用 List 内部数组

## 集合选择决策

- 键值映射 → `Dictionary<K,V>`
- 去重 → `HashSet<T>`
- FIFO → `Queue<T>`；LIFO → `Stack<T>`
- 频繁中间插入 → `LinkedList<T>`
- 默认首选 → `List<T>`

## 与已有页面关联

- [[CSharp值类型性能]] — struct 在 List 中的连续存储与缓存友好性
- [[CSharp内存GC]] — List 扩容导致的 GC 压力与 struct 优化
- [[CSharp异步模型]] — 并发集合（ConcurrentQueue/Channel）在异步场景的应用
- [[CSharp序列化IO]] — 集合的 XML 序列化与 IXmlSerializable

## 来源

- [[csharp-collections-摘要]]
