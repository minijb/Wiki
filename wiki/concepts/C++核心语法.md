---
title: "C++ 核心语法"
type: concept
updated: 2026-06-02
tags: [cpp, stl, move-semantics, memory, smart-pointers]
aliases: [C++语法, C++核心]
---

# C++ 核心语法

C++ 核心语法速查，从移动语义到 STL 容器全景，从智能指针到虚表内存布局。核心原则：理解底层实现以做出正确的性能和正确性决策。

## 移动语义

移动操作将资源所有权从一个对象转移到另一个对象，避免不必要的深拷贝。关键要素：

- **右值引用** `T&&` 绑定临时对象或 `std::move` 标记的左值
- 移动构造/赋值**必须标记 `noexcept`**，否则 STL 容器扩容时退化为拷贝

```cpp
class Buffer {
    char* data;
public:
    Buffer(Buffer&& other) noexcept
        : data{ other.data } { other.data = nullptr; }
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) { delete[] data; data = other.data; other.data = nullptr; }
        return *this;
    }
};
```

`std::forward<T>` 在模板中保持参数的值类别实现完美转发。

## STL 容器选择

| 需求 | 首选容器 | 原因 |
|------|---------|------|
| 快速查找 | `unordered_set` / `unordered_map` | O(1) 哈希查找 |
| 有序遍历 | `set` / `map` | O(log n) 红黑树 |
| 连续内存 / 随机访问 | `vector` | 缓存友好 |
| 双端快速插入删除 | `deque` | 分段连续，两端 O(1) |
| 任意位置插入删除 | `list` | 双向链表 |
| 大根堆 / 小根堆 | `priority_queue` | 适配器，默认最大堆 |
| header-only 依赖传递 | INTERFACE 库 | 仅传递编译选项和 include 路径 |

`unordered_set` / `unordered_map` 需自定义哈希时，可特化 `std::hash<T>` 或传入自定义哈希函数对象。

## 智能指针

- `unique_ptr`：独占所有权，不可拷贝，可移动。析构自动 delete。推荐 `make_unique`
- `shared_ptr`：共享所有权，引用计数归零时 delete。`make_shared` 一次分配（对象+控制块）
- `weak_ptr`：不增加引用计数，`lock()` 返回 `shared_ptr`。专门用于打破循环引用

## 内存管理

`malloc` 分配的是虚拟内存，首次访问才触发缺页中断映射物理内存。小内存（<128KB）通过 `brk()` 在堆顶移动，`free` 后不归还 OS（缓存在 malloc 内存池）。大内存通过 `mmap()` 分配，`free` 后立即归还。

STL 分配器两级策略：≤128B 走内存池 + 自由链表，>128B 直接 `malloc`/`free`。

## 虚表与对象模型

- **vtable**：编译期在只读数据段生成，包含虚函数地址 + type_info
- **vptr**：构造函数执行时动态绑定到对象头部（64 位下 8 字节）
- 空类大小 1 字节（不同对象必须不同地址）
- 非虚成员函数、静态成员、构造函数**不在**对象中（代码段共享）
- 多继承含多张虚表，指针转换时 this 可能调整偏移
- 虚继承引入 vbptr（虚基类表指针）维护虚基类偏移

## 编译过程

```
.cpp/.h → 预处理（宏展开、#include）→ 编译（C++→汇编）→ 汇编（.o/.obj）→ 链接（符号解析）
```

## 关联页面

- [[sources/cpp-core-syntax-摘要|C++ 核心语法 来源摘要]]
- [[concepts/CSharp值类型性能|C# 值类型性能]] — struct/class 内存布局对比
