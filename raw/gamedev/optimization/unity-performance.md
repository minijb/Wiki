---
title: Unity 性能优化
type: source
updated: 2026-06-02
tags:
  - unity
  - optimization
  - performance
---

# Unity 性能优化

## C# 语言层面优化

性能优化从来不是从 Profiler 开始的——它从你写下的第一行代码开始。C# 层面的选择决定了内存分配模式、GC 压力以及 CPU 缓存命中率，而这些因素是渲染优化和架构调整无法补救的。本节覆盖三个最容易被忽视但影响最深远的语言层面技术：集合预分配、内存布局策略，以及装箱的消除。

### 集合预分配

C# 的标准集合类型——`List<T>`、`Dictionary<TKey, TValue>`、`HashSet<T>`——内部使用数组存储元素。当元素数量超过当前容量时，集合会分配一个更大的新数组，将旧数据拷贝过去，然后丢弃旧数组。这个"丢弃"操作在托管堆上留下一块不可达的内存，等待 GC 回收。

在热路径上，这种隐式扩容是 GC 分配的隐藏大户。一个在 `Update` 中每帧创建并填充 50 个元素的 `List<int>`，默认初始容量为 0（或 4，取决于构造方式），可能经历 4-5 次扩容，每次扩容产生一个被遗弃的数组。50 帧下来就是 200+ 次额外分配。

解决方案极其简单却经常被忽略：**在构造集合时传入预估的容量参数**。

```csharp
// 糟糕：默认容量，多次扩容
var list = new List<Vector3>();
for (int i = 0; i < 500; i++) { list.Add(positions[i]); }

// 正确：预分配容量
var list = new List<Vector3>(500);
```

对于 `Dictionary`，容量参数不仅预分配内部桶数组，还影响哈希碰撞的概率——容量越大，负载因子越低，查找越快。但要注意容量必须是质数或接近质数（Dictionary 内部会自动调整为质数），直接传入 1000 实际可能分配 1009 个桶。

一个更微妙的模式是**复用集合**。将 `List<T>` 提升为成员变量，使用前调用 `Clear()`，而非在方法内部反复 `new`。`Clear()` 不释放内部数组——它只是将内部 `_size` 字段置零——因此后续 `Add` 调用不需要任何分配，直到再次超过容量。

### 内存布局与缓存

理解 C# 中 `struct` 和 `class` 的区别是性能优化的分水岭。`class` 实例分配在托管堆上，变量持有的是引用（指针）。`struct` 实例可以分配在栈上、或内联在另一个对象/数组内部。这个区别对 CPU 缓存行为的影响远比"分配快慢"更重要。

**引用的连续性不等于内存的连续性。** 如果你有一个 `List<SomeClass>`，列表内部的数组存储的是指向堆上各对象的指针。这些指针在数组中是连续的，但它们指向的对象散布在堆的各处。遍历这个列表时，CPU 预取器无法预测下一个对象的位置，导致大量的缓存未命中。

相比之下，`List<SomeStruct>` 将结构体的所有字段内联存储在数组的连续内存中。遍历时，CPU 可以预取连续的内存行，缓存命中率大幅提升。这就是"数据导向设计"（Data-Oriented Design）的核心：**按访问模式组织数据，而非按抽象概念**。

更进一步，考虑**结构体数组（Structure of Arrays, SoA）**模式。假设你有一个包含位置、速度、生命值的实体：

```csharp
// Array of Structures (AoS)：每个结构体包含全部字段
struct Entity {
    public Vector3 Position;
    public Vector3 Velocity;
    public float Health;
}
Entity[] entities = new Entity[10000];

// Structure of Arrays (SoA)：每个字段一个独立数组
Vector3[] positions = new Vector3[10000];
Vector3[] velocities = new Vector3[10000];
float[] healths = new float[10000];
```

当只需要更新位置时，SoA 模式让 CPU 只加载 `positions` 和 `velocities` 数组到缓存行中，完全不需要触及 `healths` 的数据。AoS 模式下，每个缓存行（通常 64 字节）同时包含位置、速度和生命值，大量带宽浪费在不需要的数据上。对于 Unity 中成千上万个实体的批量更新（如粒子系统、群集模拟），SoA 模式可以带来数倍的吞吐量提升。

**内存对齐**也是影响 SIMD 和缓存的关键因素。`Vector3` 在 C# 中是 12 字节（3 个 float），但某些平台和编译器会将其对齐到 16 字节边界。将频繁访问的结构体标记为 `[StructLayout(LayoutKind.Sequential)]` 并手动控制字段顺序（将最常访问的字段放在前面，将相同大小的字段组合在一起以减少填充），可以进一步优化缓存利用率。

### 装箱与泛型

装箱（Boxing）是 C# 中价值类型到引用类型的隐式转换过程。将一个 `int` 赋值给 `object` 变量时，CLR 在堆上分配一块内存，将 `int` 的值拷贝进去，附加上类型对象指针和同步块索引，然后返回指向这块堆内存的引用。整个过程涉及：

1. **堆分配** — 每个装箱操作在托管堆上分配 12-24 字节（取决于平台位数）
2. **内存拷贝** — 将值类型的数据从栈/寄存器拷贝到堆
3. **GC 压力** — 装箱对象成为 GC 追踪的目标

在 Unity 中，最常见的意外装箱场景包括：

- **将值类型传入接受 `object` 参数的方法**，如 `Debug.Log(someInt)` 的旧版重载、`string.Format("{0}", someFloat)`、以及任何自定义的 `object` 参数方法。
- **值类型实现接口并通过接口调用**。`struct MyData : IComparable<MyData>`，当通过 `IComparable<MyData>` 类型的变量调用 `CompareTo` 时，值类型会被装箱。只有通过具体类型调用时才走虚方法分派而不会装箱。
- **将值类型添加到非泛型集合**，如 `ArrayList`、`Hashtable`。每 `Add` 一次就装箱一次。
- **将值类型赋给 `object`、`ValueType` 或 `Enum` 类型的变量**。
- **Unity 的 `Coroutine`**：`yield return null` 不装箱（null 不是值类型），但 `yield return 0`、`yield return new WaitForSeconds(1f)` 会装箱。应使用缓存的 `WaitForSeconds` 实例和 `yield return null`。

消除装箱的**第一原则是使用泛型**。`List<int>` 替代 `ArrayList`，`Dictionary<int, string>` 替代 `Hashtable`，`IComparer<T>` 替代 `IComparer`。泛型在 JIT 编译时为每种值类型生成专用代码，完全消除装箱。

**第二原则是避免不必要的接口分派**。如果结构体实现了 `IEquatable<T>`，调用方应通过泛型约束 `where T : IEquatable<T>` 来调用，而非通过接口变量。Unity 的 `EqualityComparer<T>.Default` 会在运行时检测 `IEquatable<T>` 实现并使用泛型路径，但存在一些边缘情况。

**第三原则是针对 Unity 特有的坑**：避免在 `Update` 或 `FixedUpdate` 中使用 `UnityEngine.Object` 的空值比较。`UnityEngine.Object` 重载了 `==` 运算符，其底层调用 `CompareBaseObjects` 会进行原生互操作，比纯 C# 引用比较慢一个数量级。在热循环中，缓存为 C# 的 `null` 检查或使用 `ReferenceEquals` 可以显著减少开销。

## 字符串优化

字符串是 Unity 中最隐蔽的 GC 来源。C# 的字符串是**不可变的**（immutable）：一旦创建，其内容永不可更改。每一次拼接、格式化、大小写转换都会在堆上生成一个新的字符串对象，旧的字符串变成垃圾等待回收。在 UI 更新、日志输出、调试信息、排行榜生成等场景中，字符串分配可以轻易压垮 GC，导致每帧数十毫秒的停顿。

### 不可变性与 GC

字符串不可变性的最直接后果是**拼接即分配**。

```csharp
string result = "Score: " + score + " / " + maxScore;
```

这一行代码创建了至少 3 个临时字符串对象：`"Score: " + score` 的结果、该结果与 `" / "` 拼接的结果、以及最终与 `maxScore` 拼接的结果。在 `Update` 或 UI 刷新回调中，这每帧产生数十到数百字节的垃圾，在高频调用下迅速累积。

`StringBuilder` 是传统的解决方案——它内部维护一个可变的 `char[]` 缓冲区，拼接操作写入缓冲区而非创建新对象。但 `StringBuilder` 本身有初始分配成本，且其内部缓冲区同样存在扩容问题。对于固定格式的字符串，更好的方案是完全避免运行时拼接。

### 缓存策略

对于有限取值范围的格式化字符串，**预计算并缓存**是最优方案。典型场景：显示 0-100 的百分比文本、0-9999 的分数文本、固定枚举值的名称映射。

```csharp
// 构建时一次性生成所有可能的字符串
static readonly string[] ScoreCache = new string[10001];
static void InitCache() {
    for (int i = 0; i < ScoreCache.Length; i++)
        ScoreCache[i] = $"Score: {i}";
}

// 运行时零分配
void UpdateScoreDisplay(int score) {
    displayText.text = ScoreCache[score];
}
```

更通用的模式是使用 `Dictionary<int, string>` 作为 LRU 风格的缓存。当请求一个未缓存的格式化结果时，计算并存储；后续相同请求直接命中缓存。这种方案在值域较大但实际使用值较集中时效果最好（例如显示玩家的金币数量——大部分时间在 0-10000 之间，偶尔出现大额数字）。可以设置缓存上限，超出时清除最早或最少使用的条目，防止无限制的内存增长。

组合缓存策略进阶：当需要拼接多个动态值时，先尝试在缓存中查找组合结果。定义一种复合键（如 `(int a, int b)` 的哈希值），未命中时再执行拼接并缓存。

### unsafe 内存操作

当缓存不可行——例如动态用户输入、网络消息、实时生成的文本——可以进入 unsafe 代码绕过字符串分配。

C# 提供 `fixed` 语句固定托管对象在内存中的位置（防止 GC 移动），然后通过 `char*` 指针直接操作内存。配合 `stackalloc` 在栈上分配临时缓冲区，可以实现完全零堆分配的字符串构造：

```csharp
unsafe string BuildScoreString(int score, int maxScore) {
    const int maxLen = 64;
    char* buffer = stackalloc char[maxLen];
    int pos = 0;

    // 手动写入 "Score: "
    buffer[pos++] = 'S'; buffer[pos++] = 'c'; buffer[pos++] = 'o';
    buffer[pos++] = 'r'; buffer[pos++] = 'e'; buffer[pos++] = ':';
    buffer[pos++] = ' ';

    // 写入 score（整数转字符串，手动实现避免分配）
    pos += IntToChars(score, buffer + pos);

    buffer[pos++] = ' '; buffer[pos++] = '/'; buffer[pos++] = ' ';

    pos += IntToChars(maxScore, buffer + pos);

    return new string(buffer, 0, pos);
}
```

`stackalloc` 在栈上分配，函数返回时自动释放，完全不经过托管堆。`new string(char*, int, int)` 构造函数是唯一在堆上分配目标字符串的地方——但这是你本来就需要的最终产物，而非临时垃圾。

这种技术需要精细的边界检查和长度管理，不适合所有场景。实践中，**混合策略**效果最好：优先使用缓存，缓存未命中时回退到 `StringBuilder`（已预热/复用），仅在最高频路径上使用 unsafe 代码。在 Unity 中，`string.Format`、`StringBuilder.AppendFormat` 以及 `Debug.Log` 的格式化重载在内部使用 `StringBuilderCache`，这是一个线程静态的 `StringBuilder` 池，已经内置了复用优化。

## 渲染优化

渲染管线是大多数 Unity 项目的性能瓶颈所在。CPU 端提交绘制命令的开销与 GPU 端执行着色器的工作之间需要精妙的平衡。理解 Draw Call 的本质和各种批处理技术的适用场景，是每个 Unity 开发者的必修课。

### Draw Call 原理

Draw Call 是 CPU 向 GPU 发出的绘制指令。在 OpenGL 上，一个 Draw Call 对应一次 `glDrawElements` 或 `glDrawArrays` 调用；在 Direct3D 上，对应 `DrawIndexed` 或 `Draw`；在 Vulkan/Metal 上，对应 `vkCmdDrawIndexed` 或 `MTLRenderCommandEncoder.drawIndexedPrimitives`。无论 API 如何演进，核心概念不变：CPU 准备渲染状态（着色器、材质、缓冲区绑定、混合模式、深度测试等），然后提交绘制命令。

Draw Call 的昂贵不在于 GPU 端——GPU 擅长并行处理大量几何体。昂贵的是 **CPU 端的渲染状态设置**。每次 Draw Call 切换时，CPU 必须验证当前绑定的着色器、纹理、常量缓冲区，计算并上传变换矩阵，刷新命令缓冲区。在传统渲染管线中，数千个 Draw Call 可以让 CPU 每帧花费 5-15 毫秒仅用于状态管理，留给游戏逻辑的时间所剩无几。

减少 Draw Call 的核心策略是**批处理（Batching）**：将多个使用相同材质、相同渲染状态的网格合并为一个 Draw Call 提交。Unity 提供了四种批处理机制，各有权衡。

### 静态批处理

静态批处理（Static Batching）适用于场景中标记为 `Static` 的游戏对象。在构建时或运行时，Unity 将共享同一材质的静态网格合并为一个巨大的顶点缓冲区和索引缓冲区，存储在 GPU 显存中。

值得注意的是，**静态批处理不总是减少 Draw Call**。当一个合并后的网格包含超过 65,535 个顶点（16 位索引上限）时，Unity 会将其拆分为多个 Draw Call。静态批处理的真正价值在于**消除渲染状态切换**：所有合并后的网格共享同一个缓冲区绑定，CPU 不需要在对象之间重新绑定 VB/IB，大幅减少状态设置开销。

代价是内存：合并后的网格需要额外的显存存储。对于大量小物件（如场景装饰），这是值得的；对于已经是巨型网格的物件（如地形），反而不划算。还要注意的是，标记为 Static 的物体在运行时不能移动、旋转、缩放——这与"静态"的语义一致。

### 动态批处理

动态批处理（Dynamic Batching）是 Unity 在运行时自动对小网格进行的批处理。触发条件是网格的顶点属性总数（顶点数 × 每顶点属性数）小于一定阈值，且多个渲染器使用相同的材质。Unity 在每帧将这些网格的顶点动态变换到世界空间，合并到一个缓冲区中，然后一次性提交。

动态批处理看似"免费"，但在现代设备上可能成为陷阱。**CPU 端的顶点变换成本有时高于 Draw Call 本身的开销**。在移动平台上，动态批处理可以显著减少 Draw Call，但需要权衡 CPU 工作时间。Unity 官方建议在移动平台启用，在桌面/主机平台谨慎评估。

限制条件也颇为严格：只支持包含少于 900 个顶点属性的网格（在 2017.3+ 版本中；早期版本更严格）。使用多 Pass 着色的对象无法批处理。接收实时光照阴影的对象无法批处理——阴影会打断批处理流程。

### SRP Batcher

SRP Batcher 是 Scriptable Render Pipeline（URP/HDRP）引入的革命性优化。它不合并网格，而是重组渲染状态提交的方式。

传统批处理的核心假设是"相同材质 = 无状态切换"。SRP Batcher 将"材质数据"和"对象数据"分离为两类 GPU 常量缓冲区（CBuffer）：

- **引擎 CBuffer（Per-Object CBuffer）**：包含每个对象的变换矩阵、光照探头系数等引擎内置数据。每个对象不同，但格式固定。
- **材质 CBuffer（Per-Material CBuffer）**：包含纹理偏移、颜色、着色器全局参数等材质数据。每个材质不同，但不会每帧变化。

SRP Batcher 将材质 CBuffer 持久化在 GPU 显存中。渲染时，对于同一着色器变体的材质序列，Unity 只需要更新每个对象的 Per-Object CBuffer（一个固定大小、无变化的格式），而无需重新上传整个材质数据。这使得渲染循环在 GPU 端以极低的 CPU 开销运行——CPU 只是不断地写入变换矩阵到已固定的缓冲区中。

**前提条件**：所有参与 SRP Batcher 的材质必须使用兼容的着色器变体。着色器必须在 HLSL 代码中显式声明 `CBUFFER_START(UnityPerMaterial)` 将所有材质属性包裹在常量缓冲区中。URP 和 HDRP 的内置着色器已满足此要求，但自定义着色器需要手动适配。

SRP Batcher 的适用场景是大量使用不同材质、但共享着色器变体的对象——这正是静态/动态批处理无法处理的情况。配合 GPU Instancing，可以覆盖几乎所有的渲染场景。

### GPU Instancing

GPU Instancing 适用于大量完全相同（相同网格 + 相同材质）的物体——草、树木、子弹、人群。GPU 在硬件层面循环执行同一个 Draw Call，每次迭代使用不同的实例数据（如变换矩阵数组），CPU 只需要提交一次绘制命令。

Unity 的实现通过 `MaterialPropertyBlock` 在运行时注入每个实例的属性差异（如颜色变化），同时保持共享材质的批处理能力。着色器需要 `#pragma multi_compile_instancing` 并在顶点着色器中访问 `unity_InstanceID` 获取当前实例索引。

GPU Instancing 在移动平台的 OpenGL ES 3.0+ 和 Metal 上原生支持；在旧版设备上回退为 CPU 端的多次 Draw Call。DrawMeshInstanced 和 DrawMeshInstancedIndirect 提供了更灵活的控制——后者允许 GPU 驱动的实例数量，适合实现视锥剔除后的动态实例化。

## UI 优化

Unity UI（uGUI）系统的性能问题源于其**重建机制**。当 Canvas 下的任何 UI 元素发生改变——变换、颜色、文本内容、层级——整个 Canvas 都需要重建几何体（ReconstructGeometry）。这个过程涉及为所有可见的 UI 元素生成顶点和三角形，然后上传到 GPU。

### OnEnable 与网格重建

`OnEnable` 是 UI 性能问题的重灾区。当 UI 面板通过 `SetActive(true)` 激活时，Canvas 检测到新的渲染器组件被启用，触发一次完整的网格重建。如果面板包含 `Text` 组件，问题会严重恶化：`Text` 组件使用 FreeType 字体渲染引擎在 CPU 上生成字形几何体，涉及字形的顶点分配、纹理图集查询和字距调整。这个过程会产生大量的临时分配（顶点缓冲区、索引缓冲区、中间字符信息数组），全部在 `OnEnable` 路径中执行，最终全部变成 GC 垃圾。

**核心对策**：

1. **使用 CanvasGroup 替代 SetActive**：将 CanvasGroup 的 `alpha` 设为 0 并关闭 `interactable` 和 `blocksRaycasts`，而不是频繁启用/禁用物体。CanvasGroup 的 alpha 变化不会触发网格重建——GPU 在着色器中处理透明度。射线检测关闭后，UI 事件系统跳过该面板。

2. **将 UI 移动到屏幕外**：对于需要频繁切换但内部布局不变的 UI（如浮动提示、暂停菜单），将其 RectTransform 的 `anchoredPosition` 移到视口外（如 `(5000, 5000)`），而非禁用/启用。渲染管线在裁剪阶段跳过不可见的 UI，不产生任何额外 Draw Call。

3. **分离动态和静态 Canvas**：将频繁更新的 UI（生命值、计时器、聊天）放在独立 Canvas 上，稳定不变的 UI（背景、边框、标题）放在另一个 Canvas。子 Canvas 的更新不会触发父 Canvas 的重建。每个 Canvas 有额外的 Draw Call 开销，但相比避免全局重建的收益，这个代价微不足道。

4. **避免 `LayoutGroup` 的连锁重建**：`VerticalLayoutGroup`、`HorizontalLayoutGroup`、`ContentSizeFitter` 在每次子元素变化时触发布局重建——从子元素向上冒泡到布局组件，再向下驱动所有子元素的重新排列。对于包含数十个子元素的布局，这是一个 O(n²) 的级联。解决方案：在变化前禁用布局组件，手动设置子元素位置，变化完成后重新启用。

## Profiler 使用

性能优化的前提是测量。Unity Profiler 提供了从 CPU 到 GPU、从内存到音频的全套分析工具，但它的 API 本身也需要谨慎使用，以免引入测量偏差。

### BeginSample/EndSample

`Profiler.BeginSample` 和 `Profiler.EndSample` 在 Profiler 的时间线中创建一个自定义的标记区域，用于精确测量特定代码段的耗时。它们在 Profiler 未连接或未录制时开销极低（仅为一次函数调用和提前返回），但在录制时会将时间数据写入内部缓冲区。

```csharp
Profiler.BeginSample("AI.PathFinding");
PathFindAllAgents();
Profiler.EndSample();
```

标记名称必须是字符串。如果使用了 `string.Format` 或拼接来动态生成标记名称（如 `Profiler.BeginSample($"AI.PathFinding.{agentCount}")`），每次调用都会在托管堆上分配新的字符串，在热路径上产生显著的 GC 压力。**在标记中使用字面量字符串或预定义的常量字符串变量**，避免任何运行时分配。

嵌套的 BeginSample/EndSample 在 Profiler 视图中以层级结构显示，对应调用堆栈的深度。这使得定位性能瓶颈时可以逐层深入：从"AI 总耗时"深入到"寻路"再到"A* 启发式计算"。但要注意过深的嵌套（超过 6-8 层）会显著增加 Profiler 本身的开销，混淆测量结果。

### 封装与条件编译

将 Profiler 标记封装为辅助方法，结合条件编译，可以在不修改调用代码的情况下完全消除发布版本中的 Profiler 开销：

```csharp
public struct ProfilerScope : IDisposable {
#if ENABLE_PROFILER
    public ProfilerScope(string name) => UnityEngine.Profiling.Profiler.BeginSample(name);
    public void Dispose() => UnityEngine.Profiling.Profiler.EndSample();
#else
    public ProfilerScope(string name) { }
    public void Dispose() { }
#endif
}

// 使用 using 语句自动配对
void UpdatePath() {
    using var _ = new ProfilerScope("AI.UpdatePath");
    // ... 核心逻辑
}
```

`IDisposable` 结构体在 `using` 语句中不装箱（C# 编译器对 `IDisposable` 的 using 模式有特殊处理，当变量类型是结构体且明确实现 `IDisposable` 时不会装箱）。发布构建中 `ENABLE_PROFILER` 未定义时，整个方法体为空，JIT 编译器会将其完全内联消除为零开销。

对于更精细的控制，可以使用 `Profiler.EmitFrameMetaData` 在每帧记录自定义数据（如当前场景名称、活跃敌人数量），这些数据在 Profiler 的帧详情视图中可见，帮助将性能数据与游戏状态关联起来。

## 通用优化策略

以下优化策略横跨渲染、物理、动画和资源管理多个子系统。它们不总是适用——每项技术都有具体的收益和代价——但它们是每个 Unity 性能优化清单上的常客。

### LOD 与遮挡剔除

LOD（Level of Detail，细节层次）和遮挡剔除（Occlusion Culling）是"不要在看不见或看不清的物体上浪费性能"这一原则的两种实现方式。

**LOD** 根据物体到摄像机的距离切换几何精度。远景物体使用低面数网格和简化着色器，近景使用全精度。Unity 的 LOD Group 组件允许在 0%-100% 屏幕占比之间定义多个层级。关键技巧：**LOD 过渡区域**使用 Cross-Fade（交叉淡入淡出）模式，通过 `Screen-space dithering` 在像素级别混合两个 LOD 层级，避免几何体瞬间切换造成的视觉突变。Fade Mode 在 SRP 中通过 `LODFade` 着色器变量控制，在 Built-in 管线中通过 `unity_LODFade` 访问。

**遮挡剔除**在 CPU 端预先计算哪些物体被前方物体遮挡、完全不进入摄像机的视锥。Unity 使用 Umbra 中间件进行静态遮挡剔除：在烘焙阶段，Umbra 将场景体素化，构建可见性表格；运行时，查询摄像机所在体素的可见集，剔除不可见物体，完全不提交它们的渲染命令。用于大型室内场景（走廊、房间）效果拔群；在开阔的室外场景收益有限。

**动态遮挡剔除**是新版 Unity（2023.1+）引入的功能，不依赖烘焙，在运行时使用上一帧的深度缓冲做 Hierarchical Z 测试。适用于动态物体和无法在编辑器中预计算的场景。代价是额外的 GPU 计算和 CPU 读取回 GPU 数据（或使用 Compute Shader 间接绘制）。

### 物理优化

Unity Physics（基于 PhysX 4.1）的性能瓶颈通常在以下几个维度：

**降低模拟频率**：`Time.fixedDeltaTime` 默认 0.02 秒（50 Hz）。对于不需要高精度物理反馈的游戏（如回合制策略游戏），可以将其提升到 0.033 秒（30 Hz）甚至 0.05 秒（20 Hz）。这会降低物理精度，但 CPU 负载线性下降。注意，降低频率后 `FixedUpdate` 的调用次数同步减少，需要确保游戏逻辑不依赖高频率的物理回调。

**非分配物理查询**：`Physics.SphereCast`、`Physics.OverlapSphere`、`Physics.Raycast` 等传统 API 会为结果分配数组。使用对应的 NonAlloc 版本——`Physics.SphereCastNonAlloc`、`Physics.OverlapSphereNonAlloc`——配合预分配的静态 `Collider[]` 缓冲区：

```csharp
static Collider[] hitBuffer = new Collider[64];
int count = Physics.OverlapSphereNonAlloc(center, radius, hitBuffer);
for (int i = 0; i < count; i++) { /* process hitBuffer[i] */ }
```

NonAlloc API 返回命中的实际数量而非数组，零 GC 分配。

**减少 Rigidbody 和 Collider 数量**：休眠的 Rigidbody（未受力、未移动）不消耗 CPU，但每个活跃的 Rigidbody 在每个物理步进中都需要碰撞检测和约束求解。避免给静态障碍物添加 Rigidbody——它们只需要 Collider。使用复合 Collider（Compound Collider）替代多个独立 Collider。在大量碎片/粒子物理场景中，使用 ECS + Unity Physics（新版 DOTS Physics）而非 GameObject + Rigidbody，可以实现数量级的性能提升。

**碰撞矩阵层级过滤**：在 `Edit > Project Settings > Physics > Layer Collision Matrix` 中精细配置哪些层之间需要碰撞检测。默认全选矩阵意味着 N 个层之间进行 N² 的碰撞对检查。合理关闭不需要的层对（如"拾取物"与"拾取物"之间）可以显著减少碰撞检测对数。

### 动画优化

Unity 的 Animator 组件基于 Mecanim 状态机，性能开销来自三个方面：

**避免复杂缩放动画**：顶点缩放（Scale 变换）会导致 Unity 在每帧重新计算骨骼的变换矩阵和包围盒。如果缩放是固定的（非动画），将其烘焙到网格本身。动画化的缩放（如角色的呼吸缩放、压扁拉伸效果）在骨骼数量多时会成为 CPU 热点。

**优化 Animator Layer 数量**：每个 Animator Layer 独立混合，增加 CPU 计算量。如果 Layer 的权重为 0（未激活），Unity 仍然评估其状态机（检查转换条件）。将不需要的 Layer 的 `Weight` 设为 0 并在属性中标记是否评估，或使用 `AnimatorControllerPlayable` 进行更细粒度的控制。

**减少 Animator Controller 参数和状态数量**：每个 `Animator.SetFloat/SetInteger/SetBool` 调用会触发状态机重新评估转换条件。频繁的参数设置（如每帧更新速度参数）是必要的，但冗余的参数和状态会增加评估开销。大型状态机（100+ 状态）的转换评估时间以 O(n) 增长。

**Animator 剔除**：`Animator.cullingMode` 控制动画在物体不可见时的行为。`AlwaysAnimate`（默认）始终播放动画；`CullUpdateTransforms` 在不可见时跳过变换更新但仍评估状态机；`CullCompletely` 完全停止动画更新。对于非关键动画的远处角色，`CullUpdateTransforms` 是一个平衡性能和视觉的折中。

### 着色器优化

着色器代码在 GPU 上并行执行数百万次，每一条指令、每一个纹理采样的优化都会被放大到整个画面。

**精确选择数据类型**：在移动 GPU 上，`half`（16 位浮点）比 `float`（32 位）快，`fixed`（11 位定点）比 `half` 更快。在桌面 GPU 上，`half` 和 `fixed` 几乎总是被提升为 `float`，因此选择精度更大的意义在于移动平台。规则：颜色运算用 `half3/4`，纹理坐标用 `float2`，世界空间计算用 `float3/4`，Alpha 和混合因子用 `fixed`。

**避免动态分支**：GPU 以 warp/wavefront（NVIDIA 32 线程，AMD 64 线程）为单位执行。如果 warp 中的线程因 `if` 语句产生不同分支，GPU 必须串行执行两个分支，然后合并结果——分支发散（Branch Divergence）导致性能下降。在着色器中，应优先使用数学运算（如 `step`、`lerp`、`saturate`）替代 `if/else`。

**减少纹理采样**：纹理采样（`tex2D`/`Sample`）涉及显存访问和纹理缓存查找，是着色器中最昂贵的操作之一。合并多个小型纹理为图集（Atlas），减少采样次数。在可能的情况下，将纹理坐标计算移到顶点着色器中，通过 `TEXCOORD` 语义传递到片元着色器。

**无光照着色器**：对于不受场景光照影响的 UI、天空盒、后处理、粒子效果，使用 Unlit 着色器完全跳过光照计算，节省 GPU 时间。

### OnDemandRendering

`OnDemandRendering` API 允许 CPU 驱动渲染帧率，而非让渲染管线自由运行。适用于菜单界面、暂停状态、回合制游戏等不需要连续渲染的场景。

```csharp
// 将渲染频率降低到每秒 15 帧
UnityEngine.Rendering.OnDemandRendering.renderFrameInterval = 4;
// renderFrameInterval 的值表示每隔 N 个垂直同步周期渲染一帧
```

在 60Hz 显示器上，`renderFrameInterval = 4` 意味着每 4 个 vsync 周期（约 15 FPS）渲染一帧，中间帧被跳过。CPU 和 GPU 在跳过的帧中处于空闲状态，大幅降低功耗和发热——在移动设备和笔记本电脑上尤为关键。

当需要恢复全速渲染时（如玩家开始移动），设置 `renderFrameInterval = 1` 恢复 60 FPS。配合 `OnDemandRendering.willCurrentFrameRender` 可以在游戏逻辑中检测当前帧是否会被渲染，跳过不必要的视觉更新。

这个 API 2019.3 引入，在 Built-in 和 SRP 中均可使用。它与 `Application.targetFrameRate` 的区别在于：`targetFrameRate` 是"请求目标帧率"，操作系统调度可能不精确；`renderFrameInterval` 是硬件级的 vsync 对齐，空闲帧完全跳过。

## Unity GC 机制

理解 Unity 的垃圾回收器对于编写高性能代码至关重要。Unity 使用 Boehm-Demers-Weiser 垃圾回收器（BoehmGC），这是一个保守的、非代际的标记-清除（Mark-Sweep）收集器。它位于 `il2cpp/GC` 目录下，被编译进 IL2CPP 的运行时中。

### BoehmGC 架构

BoehmGC 的核心设计是**保守式扫描**（Conservative Scanning）：它将栈、寄存器、全局数据段中的每个值都当作潜在的指针来检查。如果某个值恰好指向堆中的某个已分配块的地址，该块就被标记为"存活"。这种设计的代价是可能产生**假阳性**——某个整数恰好等于堆中某个对象的地址，导致该对象无法被回收（内存泄漏）。但在实践中，假阳性的概率极低，尤其是在 64 位地址空间下。

BoehmGC 不是分代收集器（Generational GC）。.NET 的 GC 将堆分为 Gen 0/1/2，频繁回收短命对象（Gen 0）而很少触及长命对象（Gen 2）。BoehmGC 每次回收都扫描整个托管堆——没有"年轻代优先"的优化。这意味每分配一个临时对象都可能触发全局 GC，停顿时间与堆大小成正比。

当 BoehmGC 触发回收时，Unity 的主线程会暂停（Stop-the-World），所有正在运行的代码被冻结，直到 GC 完成标记、清除和（可选的）合并。对于大型项目，一次完整的 GC 可能耗时 5-30 毫秒，这足以在 60 FPS 的目标下造成掉帧。

### 分级内存管理

BoehmGC 的内存管理采用分级策略，以页面大小（Page Size, 通常 4096 字节）的一半作为阈值：**小对象** ≤ 2048 字节，**大对象** > 2048 字节。

**小对象分配**：GC 以内存块（block）为单位从操作系统申请内存，每个块由多个页面组成。块被细分为大小相同的空闲列表（Free List）条目。例如，8 字节的空闲链表、16 字节的空闲链表、32 字节……每个链表只处理特定大小的分配请求。当请求分配一个 20 字节的对象时，GC 查找 32 字节空闲链表（向上取整到最接近的链表大小），取出一个空闲槽位返回。这种**伙伴系统（Buddy System）变体**避免了外部碎片——空闲槽可以重新用于相同大小的分配，但可能产生内部碎片（32 字节的槽位只使用了 20 字节）。

**大对象分配**：对于超过半页大小的对象，GC 直接分配整数个 4K 页面。释放时，相邻的空闲页面被合并（合并）以减少碎片——两个相邻的 4K 空闲块合并为一个 8K 空闲块。这个过程防止了大型分配因碎片而失败，但合并操作本身有 CPU 成本。

**最小粒度**：BoehmGC 的所有分配都以 **Granule** 为单位对齐。32 位系统上一个 Granule 为 8 字节，64 位系统上为 16 字节。这意味着请求分配 1 个字节实际占用 16 字节（64 位），分配 17 个字节实际占用 32 字节（向上取整到 2 个 Granule）。对于数百万个小对象，这种内部碎片可能累积到显著的内存量。这就是为什么在热路径上应避免频繁分配小对象——不仅因为 GC 扫描时间，也因为实际内存占用远超预期。

### 装箱与避免策略

装箱与 BoehmGC 的交互尤为不利。装箱对象通常很小（一个 `int` 装箱后约 20 字节），属于小对象范畴，被放入 32 字节空闲链表。大量装箱会迅速填满小对象空闲链表，触发 GC。更糟的是，装箱对象彼此没有引用关系——它们是一个个孤立的叶子对象——但在标记阶段，GC 仍需要扫描每一个。

对于值类型的频繁操作（如数学计算、位置更新、颜色混合），遵循以下优先级：

1. **使用泛型集合和方法**，这是消除装箱的最直接手段
2. **在结构体上实现 `IEquatable<T>` 并谨慎使用接口约束**，防止通过接口调用时的装箱
3. **将值类型作为方法参数时避免 `object` 参数**，提供强类型的重载
4. **缓存 Unity 的协程 yield 对象**（`WaitForSeconds`、`WaitForEndOfFrame`），不要在 `StartCoroutine` 的每次调用中 `new` 新实例

对于 `UnityEngine.Object` 的特殊情况：在热循环中使用 `ReferenceEquals(obj, null)` 或 `obj is null`（C# 7.0+ 的模式匹配 null 检查）替代 `obj == null`。`is null` 生成的是 `ceq` IL 指令（纯引用比较），而 `== null` 通过 `UnityEngine.Object` 的重载运算符走原生互操作路径，在 IL2CPP 下涉及 `il2cpp_codegen_object_new` 和内部函数调用。

---

**关键性能的思维模式**：预分配而非事后扩容；栈分配而非堆分配；连续内存而非指针追逐；泛型而非装箱；测量而非猜测。性能优化是数据驱动的学科——Profiler 是你的指南针，GC 分配量是你的度量衡，帧时间是你的底线。
