---
title: "C# 值类型与 GC"
type: concept
updated: 2026-05-10
tags: [csharp, struct, boxing, gc, optimization, memory]
aliases: [CSharp值类型性能, C# Struct装箱, 值类型优化]
---

# C# 值类型与 GC

C# 值类型（struct）的性能优化核心：消除装箱、避免防御性拷贝、在合适的场景选择 `class` 而非 `struct`。

## 装箱的本质与代价

装箱是将值类型隐式转换为 `object` 或接口引用时，CLR 在堆上分配对象、复制值、返回引用的过程。代价：

- **堆分配** → Gen0 GC 压力
- **CPU 开销**：分配 + 复制
- 循环中高频装箱可导致显著 GC 停顿

## 三种装箱场景与解决方案

### 1. 未重载的虚方法

```csharp
p.GetType();  // 装箱 — Point 未重写 GetType()
typeof(Point); // 不装箱 — 编译时获取
```

重载所有从 `System.Object`/`System.ValueType` 继承的虚方法：`ToString()`、`Equals()`、`GetHashCode()`。

### 2. 作为 `object` 或非泛型接口参数

泛型消除装箱的**第一工具**。CLR 具现化模型为每个值类型生成专用机器码，接口调用变直接调用：

```csharp
void Process<T>(T value) where T : struct { }
void Compare<T>(T value) where T : IComparable<T> { }  // 泛型接口，不装箱
```

### 3. Dictionary Key 的隐藏装箱

struct 用作 `Dictionary<TKey, TValue>` 的 Key 但未实现 `IEquatable<T>` 时，每次查找退化为 `object.Equals(object)`，对 Key 装箱。实现 `IEquatable<T>` 或直接使用 `readonly record struct`（自动生成）。

## 现代优化策略

### `readonly struct` — 消除防御性拷贝

声明不可变意图，编译器在 `in` 参数和 `readonly ref` 访问时跳过防御性拷贝。可对 struct 中特定成员加 `readonly` 修饰符实现部分保证。

### `ref struct` — 最严格的装箱保护

编译器强制保证绝不装箱：不能作为 class 字段、不能被闭包捕获、不能用于 async/迭代器、不能作为泛型参数。`Span<T>` 和 `ReadOnlySpan<T>` 即 `ref struct`。

### `stackalloc` + `Span<T>` — 栈上临时缓冲区

```csharp
Span<byte> buffer = stackalloc byte[256];  // 栈分配，零 GC
```

### `IEquatable<T>` — 防止集合隐藏装箱

```csharp
public readonly struct ProductKey : IEquatable<ProductKey> { ... }
// 或直接: public readonly record struct ProductKey(...);
```

### .NET 8+ InlineArray — 固定大小栈数组

```csharp
[InlineArray(16)]
public struct FixedBuffer { private int _element0; }
```

类型安全，替代传统 `fixed` 缓冲区。

### CollectionsMarshal — 原地更新 Dictionary 中的 Struct

```csharp
ref var value = ref CollectionsMarshal.GetValueRefOrNullRef(dict, key);
value.Count++;  // 原地修改，零复制
```

### .NET 9+ JIT 逃逸分析

保守优化，当 JIT 能证明装箱对象不逃逸出方法时可能栈上分配。**不可依赖**，始终显式编写无装箱代码。

## 决策速查

| 场景 | 推荐 |
|------|------|
| 小型不可变数据（≤16-32 byte） | `readonly struct` |
| Dictionary/HashSet Key | `readonly record struct` 或 `IEquatable<T>` |
| 临时缓冲区/内存切片 | `Span<T>` + `stackalloc` |
| 零分配解析 | `ref struct` + `Span<T>` |
| 大数据（≥64 byte） | 优先 `class` — 复制成本超过分配节省 |
| 跨方法共享的可变数据 | `class` — struct 突变语义易出错 |

## 关联页面

- [[sources/csharp-struct-boxing-gc-摘要|C# Struct 装箱来源摘要]]
