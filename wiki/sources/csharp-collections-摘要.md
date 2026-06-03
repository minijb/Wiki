---
title: "CSharp集合框架-摘要"
source: "raw/cs/languages/csharp-collections.md"
type: source-summary
updated: 2026-06-02
tags:
  - csharp
  - collections
  - list
---

# CSharp集合框架-摘要

> 源文件: [[CSharp集合框架]]

## 核心内容

详尽梳理 C# 集合接口层次（IEnumerable → ICollection → IList → IReadOnlyList）、yield return 迭代器机制、`List<T>` 内部实现与翻倍扩容策略的 GC 和性能影响、预分配与 TrimExcess 优化、struct vs class 在集合中的选择依据、foreach vs for 性能对比、`Span<T>` 与 CollectionsMarshal 零分配操作、`ArrayPool<T>` 租赁模式、线程安全集合速览、IDisposable 与集合的级联释放模式。

## 关键要点

1. **接口层次**：IEnumerable（只读遍历）→ ICollection（Add/Remove/Count）→ IList（索引访问）
2. **List 扩容**：翻倍策略；小容量频繁扩容 → GC；大容量复制 → 内存峰值翻倍
3. **struct 在 List 中**：数据连续存储，GC 仅需跟踪内部数组，CPU 缓存友好
4. **预分配容量**：`new List<T>(capacity)` 避免扩容链，是最简单有效的优化
5. **Span/ArrayPool**：现代高性能场景的零分配工具
6. **IDisposable 级联**：`IEnumerator<T>` 继承 `IDisposable`；集合持有 IDisposable 元素时需级联释放

## 相关页面

- [[CSharp值类型性能]]
- [[CSharp内存GC]]
- [[CSharp异步模型]]
- [[CSharp序列化IO]]
