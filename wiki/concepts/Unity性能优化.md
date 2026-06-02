---
title: "Unity 性能优化"
type: concept
updated: 2026-05-11
tags: [unity, optimization, performance, gc, rendering, ui]
---

# Unity 性能优化

Unity 性能优化的全景指南，涵盖 C# 语言层面（集合预分配、内存布局、装箱消除）、字符串 GC 优化、渲染管线批处理、UI 重建机制、Profiler 精准测量、物理/动画/着色器优化，以及 Unity BoehmGC 内部机制。

## C# 语言层面优化

性能优化的起点是代码本身。C# 层面的选择决定了内存分配模式、GC 压力和 CPU 缓存命中率。

### 集合预分配

`List<T>`、`Dictionary<TKey, TValue>` 内部使用数组，超容量时分配新数组并拷贝旧数据，旧数组成为 GC 垃圾。热路径上反复扩容是隐藏的 GC 大户。

```csharp
// 预分配容量，消除扩容分配
var list = new List<Vector3>(500);

// 复用集合，Clear() 不释放内部数组
list.Clear();
```

`Dictionary` 容量影响哈希碰撞概率——容量越大，负载因子越低，查找越快。

### 内存布局与缓存

`List<class>` 数组存的是指针，对象散布堆各处，CPU 预取器无法预测——大量缓存未命中。`List<struct>` 数据内联连续存储，缓存命中率大幅提升。

SoA（Structure of Arrays）模式：按访问模式将字段拆分到独立数组。

```csharp
// SoA：批量更新位置时不触及 health 数据
Vector3[] positions = new Vector3[10000];
Vector3[] velocities = new Vector3[10000];
float[] healths = new float[10000];
```

每个缓存行（64 字节）只含相关数据，带宽利用率最大化。

### 装箱消除

装箱将值类型拷贝到堆上（12-24 字节分配），成为 GC 追踪目标。Unity 高频意外装箱场景：

| 场景 | 说明 |
|------|------|
| `object` 参数方法 | `Debug.Log(int)`、`string.Format` 的旧版重载 |
| 值类型接口调用 | `struct : IComparable<T>` 通过接口变量调用 |
| 非泛型集合 | `ArrayList`、`Hashtable` 每次 `Add` 装箱 |
| 协程 yield | `yield return 0`、`yield return new WaitForSeconds(1f)` |
| `UnityEngine.Object == null` | 重载运算符走原生互操作，比 `is null` 慢一个数量级 |

消除三原则：**泛型集合替代非泛型**、**泛型约束替代接口变量**、**`is null` 替代 `== null`**。

## 字符串优化

C# 字符串不可变：每次拼接产生新对象，旧字符串成为垃圾。`"Score: " + score + " / " + maxScore` 一行产生至少 3 个临时对象。

### 缓存策略

有限值域的格式化字符串，预计算并缓存：

```csharp
static readonly string[] ScoreCache = new string[10001];
// 构建时一次性生成，运行时零分配
displayText.text = ScoreCache[score];
```

值域大但实际使用集中的场景，使用 `Dictionary<int, string>` 做 LRU 缓存。

### unsafe 零分配拼接

`stackalloc char[]` + unsafe 指针操作，完全绕过托管堆：

```csharp
unsafe string BuildString(int score) {
    char* buffer = stackalloc char[64];
    // 手动写入字符...
    return new string(buffer, 0, length);
}
```

混合策略效果最好：缓存优先 → `StringBuilder`（已预热）→ unsafe 仅用于最高频路径。

## 渲染优化

### Draw Call 与批处理

Draw Call 的昂贵在于 **CPU 端渲染状态设置**，而非 GPU 执行。

| 批处理方式 | 原理 | 适用场景 | 限制 |
|-----------|------|---------|------|
| 静态批处理 | 构建时合并网格 VB/IB 到 GPU 显存 | 静态场景装饰 | 不能移动；超大网格拆分 |
| 动态批处理 | 运行时 CPU 变换顶点后合并 | 移动端小网格 | ≤900 顶点属性；多 Pass/阴影打断 |
| SRP Batcher | 材质/对象 CBuffer 分离，GPU 持久化 | 不同材质共享着色器变体 | URP/HDRP；着色器需 `UnityPerMaterial` |
| GPU Instancing | GPU 硬件循环同一 Draw Call | 大量同网格同材质（草、子弹） | 需 `#pragma multi_compile_instancing` |

### SRP Batcher 架构

将渲染数据分离为两类 CBuffer：

- **Per-Object CBuffer**：变换矩阵、光照探头——格式固定，每对象更新
- **Per-Material CBuffer**：纹理偏移、颜色——持久化在 GPU 显存中

渲染时 CPU 只需写入 Per-Object 矩阵，无需重新上传整个材质数据。

## UI 优化

### Canvas 重建机制

Canvas 下任何元素变化都会触发全局网格重建（ReconstructGeometry）。`OnEnable` + `Text` 组件是 GC 重灾区：FreeType 字形几何体生成涉及大量临时分配。

| 策略 | 做法 | 效果 |
|------|------|------|
| CanvasGroup 替代 SetActive | `alpha=0` + 关闭 `interactable`/`blocksRaycasts` | 不触发网格重建 |
| 移出视口 | `anchoredPosition = (5000, 5000)` | GPU 裁剪阶段跳过 |
| 分离动静 Canvas | 频繁更新 UI 独立 Canvas | 子 Canvas 变化不触发父重建 |
| 布局前禁用 LayoutGroup | 批量修改前禁用，完成后恢复 | 避免 O(n²) 级联重建 |

## Profiler 使用

### 零分配标记

```csharp
public struct ProfilerScope : IDisposable {
#if ENABLE_PROFILER
    public ProfilerScope(string name) => Profiler.BeginSample(name);
    public void Dispose() => Profiler.EndSample();
#else
    public ProfilerScope(string name) { }
    public void Dispose() { }
#endif
}

// 使用：结构体 using 不装箱
using var _ = new ProfilerScope("AI.PathFinding");
```

发布版本 `ENABLE_PROFILER` 未定义时，整个方法体为空，JIT 完全内联消除。标记名称必须用字面量——避免任何运行时字符串分配。

## 通用优化策略

### LOD 与遮挡剔除

- **LOD Group**：按屏幕占比切换几何精度，Cross-Fade 避免视觉突变
- **静态遮挡剔除**：Umbra 烘焙体素化可见性表，室内场景效果拔群
- **动态遮挡剔除**（2023.1+）：基于上一帧 Hierarchical Z，无需烘焙

### 物理优化

```csharp
// NonAlloc API：零 GC 分配
static Collider[] hitBuffer = new Collider[64];
int count = Physics.OverlapSphereNonAlloc(center, radius, hitBuffer);
for (int i = 0; i < count; i++) { /* process hitBuffer[i] */ }
```

- 降低 `fixedDeltaTime`（30 Hz / 20 Hz）降低物理 CPU 负载
- 碰撞矩阵层级过滤减少 N² 碰撞对
- 静态障碍物无需 Rigidbody

### 着色器优化

| 优化 | 说明 |
|------|------|
| 数据类型 | 移动端 `half` > `fixed` > `float`；桌面端均提升为 `float` |
| 避免分支 | `step`/`lerp`/`saturate` 替代 `if/else`，防止分支发散 |
| 减少采样 | 合并图集、纹理坐标计算移入顶点着色器 |
| Unlit 着色器 | UI/天空盒/后处理跳过光照计算 |

### OnDemandRendering

```csharp
// 菜单界面降至 15 FPS，hw vsync 对齐
OnDemandRendering.renderFrameInterval = 4;
// 恢复：renderFrameInterval = 1
```

比 `Application.targetFrameRate` 更精确——硬件级 vsync 对齐，空闲帧完全跳过，移动端大幅降功耗。

## Unity GC 机制

### BoehmGC 架构

Unity 使用 Boehm-Demers-Weiser GC（非 .NET 的分代 GC）：

- **保守式扫描**：栈/寄存器中的值均视为潜在指针，可能产生假阳性（64 位下极低）
- **非分代**：每次回收扫描全堆，无"年轻代优先"优化
- **Stop-the-World**：主线程完全暂停，一次完整 GC 耗时 5-30ms

### 分级内存管理

| 对象大小 | 分配方式 | 特点 |
|---------|---------|------|
| ≤ 2048 字节（小对象） | 空闲链表（伙伴系统变体） | 按粒度对齐，内部碎片（64 位最小 16 字节） |
| > 2048 字节（大对象） | 整页分配（4K 倍数） | 释放时合并相邻空闲页 |

大量小对象（如装箱后的 `int`）迅速填满空闲链表，频繁触发 GC。避免热路径上的小对象分配。

## 参见

- [[sources/Unity性能优化-摘要|来源摘要]] — 原始文件要点
- [[concepts/CSharp内存GC|C# 内存与 GC]] — 分代 GC、装箱、Dispose 模式
- [[concepts/CSharp值类型性能|C# 值类型性能]] — struct 装箱消除、ref struct、Span
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 编辑器相关优化属性
