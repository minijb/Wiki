---
title: "CSharp内存GC"
tags:
  - csharp
  - gc
  - memory
  - boxing
  - idisposable
type: concept
updated: 2026-06-02
---

# CSharp内存GC

C# 内存管理与垃圾回收的深度解析，涵盖值类型/引用类型分配、GC 分代回收、GC 模式与延迟控制、装箱拆箱优化、闭包与异步回调陷阱、资源释放模式。

## 核心概念

### 分代 GC

Gen0（新分配对象，短期存活）→ Gen1（从 Gen0 幸存）→ Gen2（长期存活对象）。代数越低回收越频繁、速度越快。Gen2 回收代价最高，通常由 Background GC 在后台线程执行以避免阻塞用户线程。

### GC 模式：Workstation vs Server

.NET GC 有两种主要模式，通过 `DOTNET_gcServer` 或 `.csproj` 配置：

| 模式 | 特征 | 适用场景 |
|------|------|----------|
| **Workstation**（默认）| 单 GC 堆，低延迟优先，与用户代码并发 | 桌面应用、UI 程序 |
| **Server** | 每 CPU 核心一个 GC 堆，高吞吐优先 | ASP.NET 服务端 |

### GC 延迟模式

通过 `GCSettings.LatencyMode` 控制：

| 模式 | 行为 | 使用场景 |
|------|------|----------|
| `Interactive`（默认）| 平衡吞吐与延迟 | 通用 |
| `Batch` | 最大化吞吐，允许更长的 GC 暂停 | 批处理、无用户交互 |
| `LowLatency` | Gen2 回收受抑制，暂停更短 | 时间敏感操作（需手动退出）|
| `SustainedLowLatency` | 长期低延迟，仅 foreground Gen2 受抑制 | 长期运行的响应式服务 |

### 装箱

值类型 → `object`/接口，堆分配 + 内存复制。三大避免策略：重载基类方法（`ToString`/`GetHashCode`/`Equals`）、泛型约束替代接口参数、统一接口提前拆箱。详见 [[CSharp值类型性能]]。

### 闭包 GC

Lambda 捕获外部变量 → 编译器生成匿名类（堆分配）→ 每次调用可能产生新对象。高频路径（如 `Update` / `FixedUpdate`）中避免创建闭包，改用 `foreach` 或缓存委托。

### 异步回调 GC 陷阱

异步图片加载中常见的竞态 + GC 问题：同时两次 `LoadImage` 同一路径 → 两个回调都执行 `cachedTextures.Add(path, tex)` → 重复 key 异常。

**修复方案**：加载列表 `HashSet<string>` + 回调队列去重，确保同一路径只发起一次异步加载，后续请求排队等待。

### Dispose 模式

`IDisposable` + `Dispose(bool)` 区分托管/非托管资源；`GC.SuppressFinalize` 避免重复清理。

## List 中的 struct vs class

在 `List<T>` 中存储百万级元素时：
- **class**：100 万个独立堆对象需要 GC 逐个扫描、移动、更新引用
- **struct**：仅 1 个堆对象（内部数组），数据连续、CPU 缓存友好

> [!warning] struct 大小限制
> struct 超过 64 字节（CPU cache line 典型大小）时，复制成本会抵消 GC 节省。大数据优先用 `class`。

## 与已有页面关联

- [[CSharp值类型性能]] — struct 在集合中的性能优势深入分析
- [[CSharp异步模型]] — ValueTask 通过值类型减少异步方法的堆分配
- [[CSharp集合框架]] — List 扩容导致的 GC 与 struct 优化实践
- [[CSharp委托特性]] — Lambda 闭包的 GC 影响与优化策略
- [[Unity性能优化]] — BoehmGC 保守式扫描与非分代回收机制，与 .NET 分代 GC 的对比及优化策略

## 来源

- [[csharp-memory-gc-摘要]]
- [[csharp-struct-boxing-gc-摘要]]
