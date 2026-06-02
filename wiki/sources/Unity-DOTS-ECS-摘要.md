---
title: "Unity DOTS 与 ECS 架构 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/gameplay/unity-dots-ecs.md"
tags: [unity, dots, ecs, job-system, burst]
---

# Unity DOTS 与 ECS 架构

## 来源

`raw/gamedev/gameplay/unity-dots-ecs.md` — 综合整理 Unity DOTS 四大支柱：Entities ECS、Job System、Burst Compiler、Mathematics

## 要点

1. **ECS 核心分离** — Entity 仅是唯一 ID（整数索引/版本号），Component 是纯数据 struct（`IComponentData`、`IBufferElementData`），System 是纯逻辑（查询 + 变换）。三者彻底分离，组合优于继承在数据层面的极致体现
2. **World 与 Archetype** — World 是 Entity 容器，每个 World 拥有独立 `EntityManager`；Archetype 是拥有完全相同 Component 类型组合的 Entity 集合，使得 EntityQuery 可在常数时间定位所有匹配 Chunk
3. **Chunk 内存布局** — Chunk 为 16 KiB 原子分配单位，同一 Chunk 内每种 Component 以连续数组存储（SoA 变体），同一 Archetype 的所有 Chunk 通过链表连接。一个 40 字节/Entity 的 Archetype 每 Chunk 可容纳约 406 个 Entity
4. **DOTS 技术栈四支柱** — Entities（ECS 运行时）、Job System（无锁多线程调度 + 自动依赖追踪）、Burst Compiler（LLVM AOT 编译 + SIMD 自动向量化）、Mathematics（`float3`/`quaternion`/`float4x4` 等 SIMD 友好类型代替 `Vector3`/`Quaternion`）
5. **Job System 安全模型** — 读写冲突检测（Safety System 运行时检查）、`NativeContainer` Dispose 追踪（引用计数 + 编辑器警告）、已调度 Job 引用的容器禁止主线程访问（须 `Complete()` 等待）
6. **Burst Compiler 核心约束** — 仅支持 Blittable 类型：禁止引用类型（`class`、`string`、`delegate`）、禁止 `try-catch`、禁止调用非 Burst 兼容托管代码。类型转换使用 `NativeArray<Vector3>.Reinterpret<float3>()` 零拷贝重解释
7. **Native 集合类型体系** — `Unity.Collections`（完整安全检查）与 `Unity.Collections.LowLevel.Unsafe`（零开销）两层；嵌套容器外 Native 内 Unsafe 允许、双 Native 禁止；生命周期通过 `Allocator.Temp`/`TempJob`/`Persistent` 控制，必须 `Dispose()`
8. **Blittable 约束与 SoA 布局** — Blittable 要求二进制布局在托管/非托管内存完全一致（禁止 `bool`/`char`/`string`/`decimal`）；SoA 将同类型 Component 连续排列，缓存行 100% 有效数据命中，配合 Burst + SIMD 可达传统 GameObject 10-30× 吞吐量

## 关联 Wiki 页面

- [[concepts/Unity DOTS 与 ECS|Unity DOTS 与 ECS]] — 概念页
- [[concepts/CSharp值类型性能|C# 值类型与 GC]] — 装箱消除、`ref struct`、零分配优化
