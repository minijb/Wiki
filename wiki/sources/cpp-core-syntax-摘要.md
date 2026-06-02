---
title: "C++ 核心语法 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [cpp, stl, move-semantics, memory, smart-pointers]
source: "raw/cs/languages/cpp-core-syntax.md"
---

# C++ 核心语法 — 摘要

来源：`raw/cs/languages/cpp-core-syntax.md`

## 概述

C++ 核心语法系统速查，涵盖从 C++11 到 C++17 的关键特性：移动语义与完美转发、STL 六大部件全景、容器选择决策、哈希自定义、智能指针、内存管理底层原理、虚表与对象布局、编译过程。

## 要点

- **移动语义**：右值引用 `T&&` 捕获临时对象，`std::move` 标记左值为右值。移动构造/赋值必须标记 `noexcept`，否则 STL 容器扩容时退化为拷贝
- **完美转发**：模板中的 `T&&`（万能引用）+ `std::forward<T>` 保持参数值类别原样传递
- **STL 容器分层**：序列容器（vector/deque/list/forward_list/array）、容器适配器（stack/queue/priority_queue）、有序关联容器（set/map 基于红黑树 O(log n)）、无序关联容器（unordered_set/map 基于哈希表 O(1)）。优先使用 unordered 除非需要有序
- **哈希自定义**：特化 `std::hash<T>` 或传入自定义哈希函数对象 + 相等比较器。组合哈希避免顺序冲突（`h1 ^ (h2 << 1)`）
- **智能指针**：`unique_ptr`（独占，不可拷贝）、`shared_ptr`（引用计数）、`weak_ptr`（打破循环引用）；推荐 `make_unique`/`make_shared`
- **内存管理**：malloc 分配虚拟内存，首次访问触发缺页中断；小内存 (<128KB) 通过 brk() 不立即归还 OS；STL 两级分配器：≤128B 走内存池+自由链表，>128B 走 malloc
- **强制转换**：`static_cast`（编译期安全）、`const_cast`（移除 const）、`dynamic_cast`（运行期向下转型）、`reinterpret_cast`（任意指针互转，危险）
- **虚表布局**：vtable 在只读数据段，vptr 在构造函数中绑定。多继承含多张虚表，虚继承引入 vbptr

## 关联页面

- [[concepts/C++核心语法|C++ 核心语法]] — 概念综合页
- [[concepts/CSharp值类型性能|C# 值类型性能]] — struct 内存布局对比
