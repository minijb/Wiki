---
title: "Unity 性能优化 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/optimization/unity-performance.md"
tags: [unity, optimization, performance]
---

# Unity 性能优化

## 来源

`raw/gamedev/optimization/unity-performance.md` — Unity 性能优化全景指南

## 要点

1. **C# 集合预分配** — `List<T>`、`Dictionary<TKey, TValue>` 构造时传入容量参数，避免内部数组多次扩容产生的 GC 压力。复用集合 + `Clear()` 替代反复 `new`，`Clear()` 不释放内部数组
2. **内存布局与 CPU 缓存** — `List<struct>` 数据内联连续存储，缓存命中率远高于 `List<class>` 的指针追逐。SoA（Structure of Arrays）模式下按需加载字段，避免缓存行浪费。`[StructLayout]` 控制对齐减少填充
3. **装箱与泛型消除策略** — 装箱三大代价（堆分配 12-24 字节、内存拷贝、GC 追踪）。Unity 高频装箱场景：`object` 参数方法、值类型接口调用、非泛型集合、协程 yield、`UnityEngine.Object` 的空值比较。消除三原则：泛型优先、避免接口分派、`is null` 替代 `== null`
4. **字符串不可变性与 GC** — 每次拼接产生临时对象，`Update` 中高频拼接快速压垮 GC。缓存优先策略：预计算有限域字符串数组、LRU 缓存字典。终极方案：`stackalloc char*` + unsafe 零堆分配拼接，混合策略最佳
5. **Draw Call 与批处理** — CPU 渲染状态设置是真正瓶颈。四种批处理：静态批处理（合并 VB/IB 到 GPU，消除状态切换）、动态批处理（运行时合并小网格，移动端优先，桌面端权衡 CPU 顶点变换成本）、SRP Batcher（材质/对象 CBuffer 分离，兼容不同材质）、GPU Instancing（GPU 硬件循环，同网格同材质大量实例）
6. **UI 重建机制优化** — Canvas 网格重建触发源。`OnEnable` + `Text` 组件是 GC 重灾区。四对策：CanvasGroup alpha 替代 SetActive、移出视口替代禁用、分离动静 Canvas、布局前禁用 LayoutGroup
7. **Profiler 精准测量** — `BeginSample`/`EndSample` 标记字符串必须用字面量避免 GC。`ProfilerScope` 结构体 + `using` + 条件编译 `ENABLE_PROFILER`，发布版本零开销消除。过深嵌套（>6-8 层）增加自身开销
8. **LOD 与遮挡剔除** — LOD Group 按屏幕占比切换几何精度，Cross-Fade 模式避免视觉突变。Umbra 静态遮挡剔除适合室内场景。2023.1+ 动态遮挡剔除基于 Hierarchical Z
9. **物理优化** — 降低 `fixedDeltaTime`（30/20 Hz）、NonAlloc API（`OverlapSphereNonAlloc` + 静态缓冲区）、减少活跃 Rigidbody、碰撞矩阵层级过滤
10. **动画与着色器优化** — `Animator.cullingMode` 剔除不可见动画。着色器：移动端 `half`/`fixed` 优先，数学运算替代 `if/else`，减少纹理采样、合并图集、Unlit 跳过光照
11. **OnDemandRendering** — `renderFrameInterval` 硬件 vsync 对齐降低渲染帧率，菜单/暂停场景大幅降功耗，比 `targetFrameRate` 更精确
12. **Unity BoehmGC 机制** — 保守式、非分代、标记-清除。全堆扫描，停顿与堆大小成正比（5-30ms）。小对象 ≤2048 字节用空闲链表（伙伴系统变体），大对象 >2048 字节整页分配。最小粒度 16 字节（64 位），小对象内部碎片显著

## 关联 Wiki 页面

- [[concepts/Unity性能优化|Unity 性能优化]] — 概念页
- [[concepts/CSharp内存GC|C# 内存与 GC]] — GC 分代回收、装箱、Dispose 模式
- [[concepts/CSharp值类型性能|C# 值类型性能]] — struct 装箱消除、ref struct、Span
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 编辑器相关属性
