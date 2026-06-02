---
title: "Unity DOTS 与 ECS 架构"
type: concept
updated: 2026-05-11
tags: [unity, dots, ecs, job-system, burst]
---

# Unity DOTS 与 ECS 架构

Unity DOTS（Data-Oriented Technology Stack）是面向高性能场景的完整技术栈，由 Entities ECS、Job System、Burst Compiler、Mathematics 四大支柱组成。其核心思想是将数据与行为彻底分离，以 SoA（Structure of Arrays）内存布局替代传统 AoS（Array of Structures），实现 CPU 缓存友好和 SIMD 自动向量化，在典型数值计算场景下可达传统 GameObject 模式的 10-30 倍吞吐量。

## ECS 核心

> [!note] ECS = Entity（标识）+ Component（数据）+ System（逻辑）

| 概念 | 职责 | 仅包含 | 不包含 |
|------|------|--------|--------|
| **Entity** | 唯一标识 | 整数 ID（索引 + 版本号） | 数据、行为、名称 |
| **Component** | 纯数据 | 值类型字段（Blittable struct） | 逻辑、方法 |
| **System** | 纯逻辑 | EntityQuery + 变换代码 | 持久状态 |

### World 与 Archetype

**World** 是 Entity 的容器——每个 World 拥有独立的 `EntityManager`，负责 Entity 的创建、销毁和 Component 增删。项目中可同时存在多个隔离的 World（主游戏 World、编辑器预览 World 等）。

**Archetype** 是拥有完全相同 Component 类型组合的 Entity 集合。添加或移除 Component 会导致 Entity 的 Archetype 迁移（数据从当前 Chunk memcpy 到目标 Archetype 的 Chunk）。Archetype 使 EntityQuery 可在常数时间内定位所有匹配的 Chunk。

### Chunk 内存布局

**Chunk** 是 ECS 内存分配的原子单位，固定为 16 KiB（16,384 字节）。每个 Chunk 只存储同一 Archetype 的 Entity 数据，同类型 Component 以连续数组排列（SoA 变体）：

```
Chunk 内部布局（Archetype 含 Component A、B、C）：

  A[0] A[1] A[2] ...    ← 连续 float3 数组
  B[0] B[1] B[2] ...    ← 连续 quaternion 数组
  C[0] C[1] C[2] ...    ← 连续 float 数组
```

单 Chunk 容量 = (16,384 − Header) / (所有 Component 大小之和)。例如 `Translation`(12B) + `Rotation`(16B) + `Velocity`(12B) = 40B 的 Archetype 每 Chunk 约容纳 406 个 Entity。

## DOTS 技术栈

| 组件 | 用途 | 核心价值 |
|------|------|----------|
| **Entities** | ECS 架构运行时 | 数据与行为分离，Chunk 连续内存，EntityQuery 常数时间筛选 |
| **Job System** | 多线程任务调度 | 无锁作业调度，自动依赖追踪，最大化 CPU 利用率 |
| **Burst Compiler** | LLVM 编译器后端 | C# 子集编译为高度优化原生代码，SIMD 自动向量化 |
| **Mathematics** | 数学库 | `float3`/`quaternion`/`float4x4` 等 SIMD 友好类型 |

### Player Loop 执行顺序

三个主 Phase：

| Phase | 时机 | 职责 |
|-------|------|------|
| **Initialization** | 游戏启动一次 | 创建 Entity、设置初始状态 |
| **Simulation** | 每帧 | Transform、Physics、Rendering 等游戏逻辑 |
| **Presentation** | Simulation 之后 | 视觉效果、动画、音频等非关键路径 |

System 通过 `[UpdateInGroup]` 归属 Phase，通过 `[UpdateBefore]`/`[UpdateAfter]` 控制同组内的执行顺序。

## Job System

Job System 提供安全的多线程任务调度。开发者声明数据和依赖，调度器负责并行执行和同步。

### Job 调度流程

1. 创建 Job struct 实例，填入数据引用
2. 调用 `Schedule()` 返回 `JobHandle`
3. 将 `JobHandle` 传递给后续依赖 Job
4. Job System 自动在前序 Job 间建立 barrier

### 安全保证

- **读写冲突检测**：多 Job 同时写入同一 `NativeContainer` 时 Safety System 抛出异常
- **Dispose 追踪**：`NativeContainer` 引用计数，未释放资源在编辑器产生警告
- **主线程隔离**：已调度 Job 引用的容器禁止主线程访问（须 `Complete()` 等待）

### Burst 编译器

基于 **LLVM** 的 AOT 编译器，用 `[BurstCompile]` 标记。核心约束：

- 仅支持 Blittable 值类型 — 禁止 `class`、`string`、`delegate`
- 禁止 `try-catch`
- 禁止调用非 Burst 兼容的托管代码

自动检测循环中的 SIMD 机会：使用 `Unity.Mathematics` 类型（`float3`、`float4`、`float4x4`）时，编译器将 4 个 float 操作合并为单条 SSE/AVX/NEON 指令。

**类型转换**：传统 `Vector3` 不能直接被 Burst 高效处理，使用 `NativeArray<Vector3>.Reinterpret<float3>()` 零拷贝重解释。

**资源管理规则**：
- 禁止从 Burst 方法返回 `NativeArray`（Dispose 追踪失效）
- 禁止按值传递 `NativeArray`（引用计数混乱），使用 `ref NativeArray<T>`

### Native 集合类型

安全性分两层：

| 命名空间 | 安全特性 | 适用场景 |
|----------|----------|----------|
| `Unity.Collections` | 完整 Dispose 追踪、线程安全检查 | 大多数 Job 场景 |
| `Unity.Collections.LowLevel.Unsafe` | 零开销，无检查 | 性能极致场景、嵌套容器 |

嵌套容器规则：`NativeList<UnsafeList<T>>` 允许，`NativeList<NativeList<T>>` 禁止（双重引用计数冲突）。

核心集合：

| 类别 | 类型 | 说明 |
|------|------|------|
| 数组 | `NativeArray<T>` | 固定大小连续数组，最常用 |
| 数组 | `NativeList<T>` | 可动态扩容 |
| 数组 | `UnsafeList<T>` | 无检查版本 |
| 映射 | `NativeHashMap<K,V>` | 并行读取安全 |
| 映射 | `NativeMultiHashMap<K,V>` | 一键多值，用于数据分组 |
| 映射 | `NativeHashSet<T>` | 哈希集合 |
| 字符串 | `NativeText` | UTF-8、可扩容、Burst 兼容的唯一字符串形式 |
| 字符串 | `FixedString32Bytes` 等系列 | 栈上固定大小字符串 |
| 位 | `NativeBitArray` | 动态大小安全位数组 |
| 其他 | `NativeReference<T>` | 单个值的容器引用 |
| 其他 | `UnsafeAtomicCounter32/64` | 并行原子累加 |

**生命周期**：所有 Native 集合须调用 `Dispose()`。分配时指定 `Allocator`：

- `Allocator.Temp` — ≤1 帧，分配最快
- `Allocator.TempJob` — ≤4 帧，自动回收
- `Allocator.Persistent` — 持久，必须手动 Dispose

## 类型约束

### Blittable 类型层级

```
Value Types（值类型）
  └─ Unmanaged Types（非托管类型）
       └─ Blittable Types（可直接 memcpy 复制）
```

**Blittable** 要求二进制布局在托管内存和非托管内存中完全一致。

| 分类 | 类型 |
|------|------|
| ✅ Blittable | `byte`/`sbyte`/`short`/`ushort`/`int`/`uint`/`long`/`ulong`/`float`/`double`/`IntPtr`/`UIntPtr`、仅含 Blittable 字段的 struct |
| ❌ 非 Blittable | `bool`（大小不定）、`char`（需编码转换）、`string`（引用类型）、`decimal`、含引用字段的 struct |

非 Blittable 类型在跨线程传递时的风险：
- `string` 的 `memcpy` 仅复制指针不复制字符数据，GC 回收后导致竞态条件和内存损坏
- `bool` 在不同平台可能为 1/2/4 字节，位模式解释不可靠
- 含引用字段的 struct 无法安全按值跨线程复制

**ECS Component 和 Job 中安全使用的类型**：所有数值原始类型、`Unity.Mathematics` 类型（`float3`、`quaternion`、`float4x4` 等）、`Unity.Collections` Native 容器、仅含上述类型的自定义 struct。

## 数据布局

### SoA vs AoS

**AoS（Array of Structures）** — 传统面向对象：

每个元素的全部字段交错存放在一起。遍历 `Position` 时，`Velocity` 和 `Health` 也被加载到缓存行，约 25% 缓存带宽浪费在无关数据上。

**SoA（Structure of Arrays）** — DOTS 采用：

每种字段独立连续数组。遍历 `Position` 时，缓存行 100% 是需要的数据，CPU 预取器可以完美预测访问模式。

ECS Chunk 内部即 SoA 变体：同类型 Component 的数据作为独立连续数组，当 System 只遍历 Component A 时，等价于遍历原始 `NativeArray<A>`。

### 性能对比

典型数值计算场景（10 万 Entity 位置更新）：

| 模式 | 缓存命中率 | 相对吞吐量 | 说明 |
|------|-----------|-----------|------|
| GameObject + Transform | ~10-30% | 1×（基准） | Transform 对象散布在堆上 |
| AoS struct 数组 | ~60-75% | 3-5× | 每个 Entity 一个 struct，无关字段占带宽 |
| SoA + Burst + SIMD | ~95%+ | 10-30× | 连续数据 + 自动向量化 |

SoA 的代价是代码可读性下降——查询一个 Entity 的完整状态需访问多个数组。DOTS 通过 `SystemAPI.Query` 和 `Aspect` 缓解此问题，让开发者以接近 AoS 的思维编写 SoA 代码。

## 参见

- [[sources/Unity-DOTS-ECS-摘要|Unity DOTS 与 ECS 来源摘要]]
- [[concepts/CSharp值类型性能|C# 值类型与 GC]] — 装箱消除、`ref struct`、零分配优化
- [[concepts/CSharp并发模型|C# 并发模型]] — C# 多线程编程基础
