---
title: Unity DOTS 与 ECS 架构
type: source
updated: 2026-06-02
tags:
  - unity
  - dots
  - ecs
  - job-system
  - burst
---

# Unity DOTS 与 ECS 架构

> 综合整理 Unity DOTS（Data-Oriented Technology Stack）四大支柱——Entities ECS、Job System、Burst Compiler、Mathematics——从核心概念到底层实现细节。

---

## ECS 核心概念

ECS（Entity-Component-System）是一种**数据导向**的软件架构模式，与 Unity 传统的 GameObject-Component 面向对象模式有本质不同。传统模式将数据和行为耦合在同一个对象中，ECS 则将三者彻底分离。

### Entity、Component、System

ECS 的三根支柱各司其职，互不僭越：

| 概念 | 职责 | 包含 | 不包含 |
|------|------|------|--------|
| **Entity** | 标识 | 唯一 ID（整数索引/版本号） | 数据、行为、名称 |
| **Component** | 数据 | 字段（值类型，Blittable） | 逻辑、方法 |
| **System** | 逻辑 | 查询 + 变换代码 | 持久状态 |

**Entity** 仅仅是一个轻量级的整数标识符，类似于数据库中的主键。它不持有任何数据，不包含任何方法，甚至没有名称（虽然可以附加 `EntityName` 组件用于调试）。Entity 的存在意义在于将不同的 Component 组合关联在一起——一个 Entity 的"类型"完全由其拥有的 Component 类型集合定义。

```csharp
// Entity 本质上只是一个 ID
Entity entity = entityManager.CreateEntity();
// 它没有"类型"，类型由 Component 组合决定
```

**Component** 是纯数据结构，只包含字段，不包含任何行为方法。在 ECS 中，Component 必须是值类型（struct）且满足 Blittable 约束——即内存布局在托管和非托管环境中完全一致。常见形式包括 `IComponentData`（通用组件）、`IBufferElementData`（动态数组组件）、`ISharedComponentData`（共享组件，用于按值分组）。

```csharp
// Component：纯数据，无行为
public struct Translation : IComponentData
{
    public float3 Value;
}

public struct Rotation : IComponentData
{
    public quaternion Value;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}
```

### 完整的 Component 类型体系

Unity Entities 1.0+ 提供了 9 种 Component 类型，覆盖从简单数据存储到高级分组、生命周期管理的全部需求：

| Component 类型 | 接口/基类 | 用途 | 关键特性 |
|:---------------|:----------|:-----|:---------|
| **Unmanaged** | `IComponentData` | 最常见的数据组件 | struct，仅 Blittable 字段，最高性能 |
| **Managed** | `class` + `IComponentData` | 存储引用类型字段 | class 类型，支持任意 C# 对象，性能较低 |
| **Shared** | `ISharedComponentData` | 按值分组 Entity | 相同值的 Entity 归入同一 Chunk，减少内存浪费 |
| **Cleanup** | `ICleanupComponentData` | 标记需清理的 Entity | 销毁 Entity 时保留，用于外部资源释放 |
| **Tag** | `IComponentData`（无字段） | 零大小的标签组件 | 不占内存，仅用于 EntityQuery 过滤 |
| **Buffer** | `IBufferElementData` | 动态数组 | 每个 Entity 可拥有变长数据，类似 `List<T>` |
| **Chunk** | `IChunkComponent` | 整个 Chunk 共享的值 | 一个 Chunk 只有一份数据，减少冗余 |
| **Enableable** | `IEnableableComponent` | 运行时启用/禁用 | 切换不触发结构变更，性能远优于 Add/Remove |
| **Singleton** | `IComponentData`（单实例） | World 内唯一实例 | 通过 `SystemAPI.GetSingleton<T>()` 直接访问 |

```csharp
// Tag Component：零大小，仅用于查询过滤
public struct PlayerTag : IComponentData { }

// Buffer Component：动态数组
public struct Waypoint : IBufferElementData
{
    public float3 Position;
}

// Enableable Component：运行时切换无结构变更
public struct IsInvincible : IComponentData, IEnableableComponent
{
    // 通过 SetComponentEnabled<T>(entity, false) 禁用
}

// Singleton Component
public struct GameSettings : IComponentData
{
    public float GameSpeed;
    public int MaxPlayers;
}
// 访问：SystemAPI.GetSingleton<GameSettings>()
```

> [!important] Cleanup Component 的用途
> 当 Entity 被销毁时，所有非 Cleanup 组件立即移除，但 Cleanup 组件保留。这使得 System 可以在下一帧检测到"需要清理的 Entity"并执行自定义清理逻辑（如释放 Native 资源、通知外部系统），之后手动移除 Cleanup 组件完成最终销毁。

### System 类型与生命周期

Unity ECS 提供 4 种 System 基类型，适应不同场景：

| System 类型 | 基类/接口 | 运行方式 | 适用场景 |
|:------------|:----------|:---------|:---------|
| **ISystem** | `struct ISystem` | Burst 兼容的非托管 System | 高性能需求，推荐首选 |
| **SystemBase** | `class SystemBase` | 托管 System | 需要访问托管数据或 GameObject |
| **EntityCommandBufferSystem** | `class` 派生自 `SystemBase` | 提供 ECB 实例 | 延迟执行结构变更，合并多个 System 的 ECB |
| **ComponentSystemGroup** | `class` | 组织 System 层级 | 控制 System 执行顺序和分组 |

System 的生命周期方法：

```csharp
public partial struct MySystem : ISystem
{
    // 创建时调用一次 — 初始化数据、缓存查询
    public void OnCreate(ref SystemState state) { }

    // 销毁时调用一次 — 释放资源
    public void OnDestroy(ref SystemState state) { }

    // 每帧调用 — 主逻辑
    public void OnUpdate(ref SystemState state) { }
}
```

> [!tip] ISystem vs SystemBase
> `ISystem`（struct）可以被 Burst 编译，性能优于 `SystemBase`（class）。DOTS 1.0+ 推荐 `ISystem` 作为默认选择，仅在需要访问 `GameObject`、`MonoBehaviour` 或 `Component` 引用时回退到 `SystemBase`。

**System** 包含所有逻辑，负责查询拥有特定 Component 组合的 Entity，并对它们执行变换。System 不持有任何持久状态——它每帧（或按需）运行，读取 Component 数据，执行计算，写回结果。System 之间通过 `[UpdateBefore]`、`[UpdateAfter]`、`[UpdateInGroup]` 属性控制执行顺序。

```csharp
// System：纯逻辑，通过 EntityQuery 筛选 Entity
public partial struct MoveSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<Translation>, RefRO<Velocity>>())
        {
            transform.ValueRW.Value += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}
```

**关键区别**：传统 `MonoBehaviour` 将 `Transform`（数据）和 `Update()`（行为）耦合在同一对象中；ECS 将它们分离为 `Translation` Component + `MoveSystem`。当需要新增行为时，只需添加 Component 并编写新 System——已有代码完全不受影响。这就是"组合优于继承"在数据层面的极致体现。

### World 与 Archetype

**World** 是 Entity 的容器和管理单元。每个 World 拥有独立的 `EntityManager`，负责 Entity 的创建、销毁、Component 的增删改查。一个项目中可以同时存在多个 World（如主游戏 World、编辑器预览 World），它们彼此隔离，互不干扰。

```csharp
// 获取默认 World
World world = World.DefaultGameObjectInjectionWorld;
EntityManager entityManager = world.EntityManager;
```

**Archetype**（原型）是 ECS 内存管理的核心概念。Archetype 是拥有**完全相同 Component 类型组合**的一组 Entity 的唯一标识。例如：

- 所有拥有 `Translation` 的 Entity 属于 Archetype A
- 所有拥有 `Translation + Rotation` 的 Entity 属于 Archetype B
- 所有拥有 `Translation + Rotation + Velocity` 的 Entity 属于 Archetype C

当你向一个 Entity 添加或移除 Component 时，该 Entity 的 Archetype 会发生变化——ECS 框架会将其数据从当前 Chunk 移动到目标 Archetype 对应的 Chunk 中。这一操作的代价取决于 Component 数据量，但在 DOTS 的线性内存布局下，它是高效的批量 memcpy。

Archetype 的引入使得 **EntityQuery** 可以在常数时间内定位到所有匹配的 Chunk。System 声明它需要哪些 Component，ECS 运行时查找包含这些 Component 的 Archetype，然后直接迭代这些 Archetype 下的所有 Chunk——不需要逐 Entity 检查。

### Chunk 内存布局

**Chunk** 是 ECS 内存分配的原子单位，固定为 **16 KiB**（16,384 字节）。每个 Chunk 只存储属于**同一个 Archetype** 的 Entity 数据。

Chunk 的内存布局经过精心设计以实现 CPU 缓存的最佳利用率：

```
┌──────────────────────────────────────────┐
│  Chunk Header (Entity 数组、元数据等)      │
├──────────────────────────────────────────┤
│  Component A 数组: [A₀][A₁][A₂]...[Aₙ]   │  ← 连续、紧密排列
├──────────────────────────────────────────┤
│  Component B 数组: [B₀][B₁][B₂]...[Bₙ]   │  ← 同上
├──────────────────────────────────────────┤
│  Component C 数组: [C₀][C₁][C₂]...[Cₙ]   │
└──────────────────────────────────────────┘
```

同一 Chunk 内，每个 Component 类型的数据以**数组形式连续存储**。这与传统的 Array of Structures（AoS）不同——传统模式下每个对象的全部字段交错存放在一起，CPU 遍历时缓存行中充满了不需要的字段。ECS Chunk 采用的是 **Structure of Arrays（SoA）** 的变体——同类型 Component 在内存中连续排列，当 System 只需要读取 `Translation` 时，它遍历的是纯粹的 `float3` 数组，缓存行 100% 命中有效数据。

一个 Chunk 能容纳的 Entity 数量取决于其 Archetype 的大小：

```
Entity 数量 = (16,384 - Header 大小) / (单个 Entity 所有 Component 大小之和)
```

例如，若一个 Entity 包含 `Translation`（12 字节）+ `Rotation`（16 字节）+ `Velocity`（12 字节）= 40 字节，减去 Header 约 128 字节后，每个 Chunk 可容纳约 406 个 Entity。当超过容量时，ECS 会分配新的 Chunk，所有 Chunk 通过链表连接，形成 Archetype 的完整 Chunk 链表。

这种设计的直接收益：**遍历 10,000 个 Entity 的 `Translation` 时，CPU 预取器可以线性预取相邻的 float3 数据，缓存未命中率极低。**在传统 GameObject 模式下，10,000 个 `Transform` 散布在堆内存各处，每次访问都是随机的指针跳转，缓存命中率可能低于 10%。

### 结构变更与同步点

在 ECS 中，以下操作会触发**结构变更（Structural Change）**，导致 Unity 重新组织 Chunk 内存：

- 创建或销毁 Entity
- 添加或移除 Component
- 设置 Shared Component 的值

结构变更的代价：Entity 数据必须在 Chunk 之间移动（memcpy），且可能触发**同步点（Sync Point）**——主线程等待所有已调度 Job 完成。同步点会暂时阻塞所有 Worker 线程，严重降低并行效率。

> [!warning] 避免在 Job 中执行结构变更
> Job 运行期间不能直接修改 Entity 结构。应在 Job 中收集变更请求，通过 `EntityCommandBuffer`（ECB）延迟到主线程执行。使用 `EntityCommandBufferSystem` 自动在帧末回放 ECB。

```csharp
// 正确：通过 ECB 延迟执行结构变更
[BurstCompile]
public partial struct SpawnSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.TempJob);

        // 在 Job 中记录变更请求
        var job = new MyJob { ECB = ecb.AsParallelWriter() };
        state.Dependency = job.Schedule(state.Dependency);
        state.Dependency.Complete();

        ecb.Playback(state.EntityManager);  // 主线程回放
        ecb.Dispose();
    }
}
```

> [!note] Enableable Component 的优势
> 启用/禁用 `IEnableableComponent` **不算**结构变更——不移动 Chunk，不触发同步点。对于频繁切换状态的场景（如 Buff 开关），Enableable 组件比 Add/Remove 组件性能优势巨大。

### World 初始化与自定义引导

默认情况下，Unity 自动创建 Default World 并加入所有 System。可通过 `ICustomBootstrap` 接口自定义引导流程：

```csharp
public class MyBootstrap : ICustomBootstrap
{
    public bool Initialize(string defaultWorldName)
    {
        // 手动创建 World 和 System
        var world = new World("MyCustomWorld");
        World.DefaultGameObjectInjectionWorld = world;

        var systems = DefaultWorldInitialization.GetAllSystems(WorldSystemFilterFlags.Default);
        DefaultWorldInitialization.AddSystemsToRoot(systems, world);
        return true;
    }
}
```

通过 Scripting Define Symbols 控制引导行为：
- `#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP`：禁用运行时和编辑器 World 的自动创建
- `#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_RUNTIME_WORLD`：仅禁用运行时 World
- `#UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP_EDITOR_WORLD`：仅禁用编辑器 World


---

## DOTS 技术栈

DOTS（Data-Oriented Technology Stack）是 Unity 面向高性能场景的完整技术栈，由四块基石组成：

| 组件 | 用途 | 核心价值 |
|------|------|----------|
| **Entities** | ECS 架构运行时 | 数据与行为分离，Chunk 连续内存，EntityQuery 快速筛选 |
| **Job System** | 多线程任务调度 | 无锁作业调度，自动依赖追踪，最大化 CPU 利用率 |

### 四种 Job 接口对比

| 接口 | 执行模式 | 核心方法 | 适用场景 | 关键点 |
|:-----|:---------|:---------|:---------|:-------|
| `IJob` | 单任务 | `Execute()` | 独立任务，整体操作数据 | 最简单，一次执行 |
| `IJobParallelFor` | 数据并行 | `Execute(int index)` | 处理大型数组，无数据依赖 | 最常用，需保证 `index` 之间无写入竞争 |
| `IJobParallelForTransform` | 数据并行 | `Execute(int, TransformAccess)` | 并行更新大量 Transform | 需 `TransformAccessArray`，避免装箱 |
| `IJobFor` | 串行循环 | `Execute(int index)` | 有数据依赖但需 Burst 优化的循环 | 非并行，`ScheduleParallel` 触发 Burst 优化 |

**选择决策树**：
- 只有一个独立任务 → `IJob`
- 处理数万个独立数据元素 → `IJobParallelFor`
- 处理数万个 `Transform` → `IJobParallelForTransform`（性能远超在 `IJobParallelFor` 中手动处理 `Transform`）
- 循环有数据依赖但需要 Burst 极致优化 → `IJobFor` + `ScheduleParallel`

**`IJobFor` 详解**：虽然 `Execute(int index)` 在单线程上按 0→count-1 顺序执行，但通过 `ScheduleParallel` 调度时，Burst 会将整个循环编译为高度优化的原生代码，性能可超过手写 `for` 循环：

```csharp
[BurstCompile]
public struct CumulativeJob : IJobFor
{
    public NativeArray<float> Output;
    public void Execute(int i)
    {
        if (i > 0)
            Output[i] = Output[i - 1] + 1; // 依赖前一个元素，必须串行
    }
}
// 调度：虽然串行执行，但 Burst 编译后性能远超普通 for
JobHandle handle = job.ScheduleParallel(outputArray.Length, 64, default);
```

### 使用注意事项

1. **Job 不应访问静态数据**：静态数据不受安全系统保护，多线程访问会导致竞态条件
2. **`ScheduleBatchedJobs` 可立即执行**：已调度的 Job 会被缓存，调用此方法立即清空缓存队列。ECS 系统已隐式清空，通常无需手动调用
3. **`ref returns` 限制**：无法直接修改 NativeContainer 中的 struct 内容，需通过临时变量：
   ```csharp
   MyStruct temp = myNativeArray[i];
   temp.memberVariable = 0;
   myNativeArray[i] = temp;
   ```
4. **`Schedule` 和 `Complete` 只能在主线程调用**
5. **`[ReadOnly]` 标记只读数据**：声明只读访问可提升并行安全性，允许多个 Reader 同时访问
6. **调试用 `Run` 替代 `Schedule`**：`Run` 在主线程同步执行，便于断点调试
7. **不要在 Job 中分配托管内存**：分配托管内存极慢且无法被 Burst 编译
8. **`[NativeSetThreadIndex]`**：获取当前执行的 Worker 线程索引
9. **`[NativeDisableContainerSafetyRestriction]`**：跳过线程安全检查（仅用于确认安全的极端场景）
10. **长 Job 拆分**：使用 `JobsUtility.JobWorkerCount` 获取 Worker 数量，将大任务拆分为与 Worker 数匹配的子任务，避免单 Worker 过载
11. **`JobWorkerMaximumCount`**：设置最大 Job Worker 线程数

### Blittable 类型层级

理解 Blittable 约束的关键是理清三个概念的包含关系：

```
值类型 (Value Types) ⊃ 非托管类型 (Unmanaged Types) ⊃ Blittable 类型
```

- **值类型**：最广的范畴。`int`、`float`、`bool`、`struct`、`enum` 等。
- **非托管类型**：值类型的子集。不含任何引用类型字段，所有字段递归为非托管类型。例如：`float3` 是非托管，但包含 `string` 字段的 struct 不是。
- **Blittable 类型**：非托管类型的更严格子集。在托管和非托管内存中**二进制表示完全一致**。例如：`int` 是 Blittable，但 `bool`（C# 1 字节 vs 原生可能 4 字节）和 `char`（2 字节 Unicode vs 1 字节 ANSI）**不是** Blittable。

> [!warning] 常见陷阱
> - `bool` 在 C# 中占 1 字节，但在某些原生平台是 4 字节 → **不是 Blittable**，使用 `byte` 或 `int` 替代
> - `Vector3` 是 `class`（引用类型） → 使用 `float3`（`Unity.Mathematics`）替代
> - `Quaternion` → 使用 `quaternion`
> - `string` → 使用 `FixedString32Bytes` / `FixedString64Bytes` 等 Burst 兼容的定长字符串

### SoA 设计模式

**Structure of Arrays（SoA）** 是面向数据设计的核心。对比两种内存布局：

| 模式 | 内存布局 | 缓存效率 | DOTS 适用性 |
|:-----|:---------|:---------|:------------|
| AoS（Array of Structures） | `[{HP,Pos}, {HP,Pos}, ...]` | 低——不用的字段污染缓存行 | ❌ 传统 OOP 模式 |
| SoA（Structure of Arrays） | `[HP₀,HP₁,...]` `[Pos₀,Pos₁,...]` | 高——遍历 HP 时缓存行 100% 命中 | ✅ ECS Chunk 就是 SoA |

SoA 模式下的数据设计：

```csharp
// ❌ AoS — 缓存不友好
public struct BadEntity { public int Health; public float3 Position; }
public NativeArray<BadEntity> entities;

// ✅ SoA — 并行数组，缓存友好
public NativeArray<int> Healths;
public NativeArray<float3> Positions;
```

ECS 的 Chunk 内存布局天然就是 SoA 变体——每个 Component 类型的数据在 Chunk 内以独立数组连续存储，System 只遍历它需要的 Component 数组，CPU 预取效率达到理论最优。

| **Burst Compiler** | LLVM 编译器后端 | 将 C# 子集编译为高度优化的原生代码，SIMD 自动向量化 |
| **Mathematics** | 数学库 | `float3`、`quaternion`、`float4x4` 等 SIMD 友好类型，替代 `Vector3`/`Quaternion` |

四者协同工作的典型流程：

1. System 在主线程上通过 EntityQuery 定位匹配的 Chunk
2. System 将每个 Chunk 的处理任务调度为 Job
3. Job System 将 Job 分配到多个 Worker 线程
4. Burst Compiler 将 Job 的 C# 代码编译为利用 SIMD 指令的原生代码
5. 每个 Worker 线程遍历 Chunk 中连续排列的 Component 数组，享受高缓存命中率

**Player Loop 执行顺序**：Unity DOTS 中 System 被组织到三个主要 Phase 中：

- **Initialization**：游戏启动时执行一次，用于创建 Entity、设置初始状态
- **Simulation**：主循环 Phase，每帧执行。包含 Transform、Physics、Rendering 等子系统组。大部分游戏逻辑在此阶段运行
- **Presentation**：渲染相关 Phase，在 Simulation 之后执行，处理视觉效果、动画、音频等非关键路径

每个 Phase 内部包含多个 `ComponentSystemGroup`，System 通过 `[UpdateInGroup]` 属性确定归属，通过 `[UpdateBefore]`/`[UpdateAfter]` 控制同一 Group 内的执行顺序。

```csharp
[UpdateInGroup(typeof(SimulationSystemGroup))]
[UpdateBefore(typeof(TransformSystemGroup))]
public partial struct MyGameSystem : ISystem { }
```

---

## Job System

Unity Job System 提供了安全的、无锁的多线程任务调度框架。它的核心设计原则是：**开发者声明数据和依赖，调度器负责并行执行和同步**。

Job 是一个实现了 `IJob`、`IJobParallelFor` 或 `IJobFor` 接口的结构体，包含需要处理的数据引用和执行逻辑。

```csharp
[BurstCompile]
public struct MoveJob : IJobParallelFor
{
    public NativeArray<float3> Positions;      // 读写
    [ReadOnly] public NativeArray<float3> Velocities;  // 只读
    public float DeltaTime;

    public void Execute(int index)
    {
        Positions[index] += Velocities[index] * DeltaTime;
    }
}
```

Job 的调度流程：
1. 创建 Job 结构体实例，填入数据引用
2. 调用 `Schedule()` 或 `ScheduleParallel()` 返回 `JobHandle`
3. 将 `JobHandle` 传递给后续依赖此 Job 的其他 Job 或 System
4. Job System 自动在有依赖的 Job 之间建立栅栏（barrier），前序 Job 完成后自动启动后续 Job

Job System 的关键安全保证：
- **读写冲突检测**：如果一个 Job 对 `NativeArray` 写入，另一个 Job 同时读取，Safety System 会抛出异常
- **Dispose 追踪**：`NativeContainer` 使用引用计数，未释放的资源会在编辑器中产生警告
- **主线程限制**：已调度的 Job 引用的 `NativeContainer` 不能在主线程上访问，除非调用 `Complete()` 显式等待

### Burst 编译器

Burst Compiler 是 Unity 基于 **LLVM** 的 AOT（Ahead-of-Time）编译器，将受限的 C# 子集（HPC#）编译为高度优化的原生机器码。

**启用方式**：在结构体或方法上添加 `[BurstCompile]` 属性：

```csharp
[BurstCompile]
public struct MyJob : IJob
{
    public void Execute() { /* ... */ }
}

// 也可以在 Job 外部使用
[BurstCompile]
public static float ComputeDistance(float3 a, float3 b)
{
    return math.distance(a, b);
}
```

**Burst 的核心限制——仅支持 Blittable 类型**：
- 不能使用引用类型（class、string、delegate）
- 不能使用 `try-catch`
- 不能调用非 Burst 兼容的托管代码
- 只能操作值类型和 NativeContainer

这些限制的收益是巨大的。Burst 生成的代码在数值计算密集场景下可以达到 C++ 级别的性能，远超标准 Mono/IL2CPP 编译结果。

**自动向量化（Auto-Vectorization）**：
Burst 利用 LLVM 的分析能力自动检测循环中的 SIMD 机会。当使用 `Unity.Mathematics` 类型（`float3`、`float4`、`float4x4`）时，编译器可以自动将 4 个 float 操作合并为单条 SIMD 指令（SSE/AVX on x64, NEON on ARM）。

```csharp
[BurstCompile]
public struct ScaleJob : IJobParallelFor
{
    public NativeArray<float3> Positions;
    public float3 Scale;

    public void Execute(int index)
    {
        // float3 的乘法会被自动向量化——3 个 float 同时乘
        Positions[index] *= Scale;
    }
}
```

循环级别的向量化同样会被自动检测。当 Burst 发现一个循环内连续的浮点运算没有数据依赖时，它会生成打包的 SIMD 指令。

**Burst Inspector**：Unity 编辑器中的 Burst Inspector 工具可以查看编译后的汇编代码（x64/ARM）、LLVM IR 中间表示，以及向量化报告。这是验证 Burst 是否成功优化了目标代码的唯一可靠方式。

**类型转换**：Unity 传统的 `Vector3` 不能直接被 Burst 高效处理（它是 `class` 包装的托管类型）。需要使用 `NativeArray<Vector3>.Reinterpret<float3>()` 进行零拷贝重新解释——该方法不复制数据，仅改变 C# 层面的类型视图，使同一块内存被解释为 `float3` 数组。

```csharp
NativeArray<Vector3> positions = new NativeArray<Vector3>(count, Allocator.TempJob);
// 零拷贝重新解释为 float3 数组
NativeArray<float3> positionsF3 = positions.Reinterpret<float3>();

[BurstCompile]
struct Job : IJobParallelFor
{
    public NativeArray<float3> Positions; // 直接使用 float3
    public void Execute(int i) { Positions[i] += 1f; }
}
```

**资源管理规则**：
- **NEVER 从 Burst 方法返回 `NativeArray`**：Burst 编译的代码不参与 C# 的 Dispose 追踪，返回的 NativeArray 会造成资源泄漏
- **NEVER 按值传递 `NativeArray` 作为参数**：虽然 `NativeArray<T>` 是 struct，按值传递会创建副本（虽然底层数据共享，但引用计数混乱）
- **推荐做法**：使用 `ref NativeArray<T>` 参数传递引用，或使用 `unsafe` 指针返回计算结果

```csharp
// 错误：按值传递
[BurstCompile]
void Bad(NativeArray<float> data) { }  // 拷贝 struct 开销

// 正确：按引用传递
[BurstCompile]
void Good(ref NativeArray<float> data) { }
```

### Native 集合类型

Unity 提供了专门为 Job System 和 Burst Compiler 设计的集合类型，位于 `Unity.Collections` 命名空间。这些集合具有内置的线程安全检查和 Dispose 追踪。

**安全性层次**：

| 命名空间 | 安全特性 | 适用场景 |
|----------|----------|----------|
| `Unity.Collections` | 完整的 Dispose 追踪、线程安全检查 | 大多数 Job 场景 |
| `Unity.Collections.LowLevel.Unsafe` | 无安全开销，无检查 | 性能极致场景、嵌套容器 |

**嵌套容器规则**：
- `NativeList<UnsafeList<T>>` ✅ 允许（外层有安全检查，内层无）
- `NativeList<NativeList<T>>` ❌ 禁止（双重引用计数冲突）

**核心集合类型一览**：

**数组类**：
- `NativeArray<T>` — 固定大小连续数组，最常用的 Native 容器。是 struct 但行为类似智能指针——拷贝仍共享底层数据，必须手动 Dispose
- `NativeList<T>` — 可动态扩容的列表
- `UnsafeList<T>` — 无安全检查的列表，用于嵌套或极致性能需求
- `UnsafePtrList<T>` — 指针列表
- `NativeStream` — 多生产者-单消费者的流式写入数据结构
- `FixedList32Bytes<T>` — 栈上分配的小型固定大小列表（最大 32 字节数据）

**映射/集合类**：
- `NativeHashMap<TKey, TValue>` — 键值对映射，支持并行读取
- `UnsafeHashMap<TKey, TValue>` — 无检查版本
- `NativeHashSet<T>` — 哈希集合
- `NativeMultiHashMap<TKey, TValue>` — 一键多值映射，常用于数据分组（如按 Chunk ID 收集结果）

**位数组**：
- `BitField32` / `BitField64` — 栈上分配的 32/64 位掩码
- `NativeBitArray` — 动态大小的安全位数组
- `UnsafeBitArray` — 无检查版本

**字符串**：
- `NativeText` — UTF-8 编码、可动态扩容的字符串，是 Burst 兼容的唯一字符串形式
- `FixedString32Bytes` / `FixedString64Bytes` / `FixedString128Bytes` / `FixedString512Bytes` / `FixedString4096Bytes` — 栈上分配的固定大小字符串

**其他**：
- `NativeReference<T>` — 单个值的 Native 容器引用
- `UnsafeAtomicCounter32` / `UnsafeAtomicCounter64` — 原子计数器，用于并行累加

**生命周期管理**：所有 Native 集合都必须在使用完毕后调用 `Dispose()`。分配时需指定 `Allocator` 类型：

- `Allocator.Temp` — 极短生命周期（≤1 帧），分配速度最快
- `Allocator.TempJob` — 单次 Job 调度内有效（≤4 帧），自动回收
- `Allocator.Persistent` — 持久分配，必须手动 Dispose

### Blittable 类型约束

Blittable 是 DOTS 和 Burst Compiler 的类型系统基石。理解它的定义和边界是正确使用 Job System 的前提。

**三层类型分类**：

```
Value Types（值类型）
  └─ Unmanaged Types（非托管类型）
       └─ Blittable Types（可直接复制类型）
```

**Blittable 的严格定义**：类型的二进制布局在托管内存和非托管内存中**完全一致**。这意味着可以直接用 `memcpy` 在两种内存之间复制而不需要任何整理（marshaling）。

- ✅ Blittable：`byte`、`sbyte`、`short`、`ushort`、`int`、`uint`、`long`、`ulong`、`float`、`double`、`IntPtr`、`UIntPtr`、以及仅包含 Blittable 字段的 struct
- ❌ 非 Blittable：`bool`（内部实现可能为 1/2/4 字节不定）、`char`（Unicode 需要编码转换）、`string`（引用类型）、`decimal`、以及任何包含引用类型字段的 struct

**为什么必须 Blittable**：Job System 将数据在线程之间传递，涉及跨线程边界的内存访问。如果传递了非 Blittable 类型：

1. **`string`** 在托管内存中是一个包含长度和指向堆上字符数组的指针的结构。直接 `memcpy` 这个结构只复制了指针，不复制字符数据本身。另一个线程通过这个指针访问字符时，原始字符串可能已被 GC 回收或移动——导致**竞态条件和内存损坏**。

2. **`bool`** 在不同平台的 C# 实现中可能是 1 字节、2 字节或 4 字节。`memcpy` 一个平台上的 bool 到另一个环境可能导致位模式解释错误。

3. **包含引用的 struct**：struct 中任何引用类型字段都意味着这个 struct 不能安全地按值跨线程复制——引用指向的对象在托管堆上，不受 Job System 的生命周期管理。

**实用结论**：在 ECS Component 和 Job 中使用以下类型是安全的：
- 所有数值原始类型（`int`、`float`、`double` 等）
- `Unity.Mathematics` 类型（`float3`、`float4`、`quaternion`、`float4x4` 等）
- `Unity.Collections` 的 Native 容器（它们内部使用指针，是 Blittable 的）
- 仅包含上述类型的自定义 struct

### SoA 数据布局

**SoA（Structure of Arrays）**与 **AoS（Array of Structures）**的选择是 DOTS 性能的基石。

**AoS 模式**（传统面向对象）：

```csharp
// Array of Structures：每个元素包含全部字段
struct Entity_AoS {
    public float3 Position;
    public float3 Velocity;
    public float Health;
}
Entity_AoS[] entities = new Entity_AoS[10000];

// 遍历更新位置时：
for (int i = 0; i < 10000; i++)
    entities[i].Position += entities[i].Velocity * dt;
```

在此循环中，每次迭代 CPU 加载一个完整的 `Entity_AoS` 到缓存行（64 字节的缓存行可能包含 1-2 个 Entity）。Position 和 Velocity 是需要的数据，但 Health 也一同被加载——它们占据宝贵的缓存空间却完全不被使用。**大约 25% 的缓存带宽浪费在无关数据上。**

**SoA 模式**（数据导向——DOTS 采用）：

```csharp
// Structure of Arrays：每个字段独立数组
NativeArray<float3> Positions = new NativeArray<float3>(10000, ...);
NativeArray<float3> Velocities = new NativeArray<float3>(10000, ...);
NativeArray<float> Healths = new NativeArray<float>(10000, ...);

// 遍历更新位置时：
for (int i = 0; i < 10000; i++)
    Positions[i] += Velocities[i] * dt;
// Healths 数组完全不需要被加载到缓存中
```

在此循环中，CPU 缓存行中 100% 是需要的数据（连续的 `float3` 值）。预取器可以完美预测数据访问模式，在前一个缓存行被处理时就开始预加载下一个缓存行。

**ECS Chunk 的 SoA 实现**：在 Chunk 内部，每种 Component 类型的数据作为一个独立的连续数组存储：

```
Chunk 包含 Entity[0..N]，拥有 Component A、B、C

  A[0] B[0] C[0]     ← 传统 AoS 的交错排列
  A[1] B[1] C[1]
  A[2] B[2] C[2]

  A[0] A[1] A[2] ...  ← ECS Chunk SoA 的分组排列
  B[0] B[1] B[2] ...
  C[0] C[1] C[2] ...
```

当一个 System 只遍历 Component A 时，它访问的内存是完全连续的 A 数组——相当于在遍历一个原始的 `NativeArray<A>`。这种线性遍历模式在现代 CPU 上极其高效，配合 SIMD 指令可以在单个时钟周期内处理多个 Entity。

**性能对比**（典型数值计算场景，10 万 Entity 位置更新）：

| 模式 | 缓存命中率 | 相对吞吐量 | 说明 |
|------|-----------|-----------|------|
| GameObject + Transform | ~10-30% | 1× (基准) | Transform 对象散布在堆上 |
| AoS struct 数组 | ~60-75% | 3-5× | 每个 Entity 一个 struct，Health 占带宽 |
| SoA + Burst + SIMD | ~95%+ | 10-30× | 连续数据 + 向量化 |

SoA 的代价是代码可读性的下降——相关数据被分散到不同数组中，查询一个 Entity 的完整状态需要访问多个数组。DOTS 通过 API 层面的改进（`SystemAPI.Query`、`Aspect`）缓解了这一问题，让开发者可以用接近 AoS 的思维编写 SoA 代码。

**替代方案——扁平数组 + 偏移**：在某些场景中，将所有数据放在一个巨大的扁平 `NativeArray<byte>` 中，通过偏移量手动索引，可以实现比 NativeArray 向量化更激进的布局优化。这是 ECS 内部实现 Chunk 的方式——Chunk 的 16 KiB 本质上就是一个扁平的字节缓冲区，每个 Component 类型在其中有固定的起始偏移和跨步（stride）。不过，这种级别的控制通常交给 ECS 框架处理即可，业务代码极少需要直接操作。
