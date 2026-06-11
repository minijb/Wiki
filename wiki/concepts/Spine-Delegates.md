---
title: "Spine Delegates"
type: concept
updated: 2026-06-11
tags: [spine, unity, delegate, callback, skeleton-animation, skeleton-graphic]
---

# Spine Delegates

Spine Unity 运行时通过四组 delegate 暴露动画管线的钩子点。`SkeletonAnimation` 提供更新阶段回调，`SkeletonRenderer` 提供渲染阶段回调，`AnimationState` 和 `TrackEntry` 提供动画生命周期事件。

## 四组 Delegate

| 分组 | 所属对象 | 数量 | SkeletonAnimation | SkeletonGraphic |
|:-----|:-----|:-----|:-----|:-----|
| 更新阶段 | `SkeletonAnimation` | 4 | 全部支持 | **不支持** |
| 渲染阶段 | `SkeletonRenderer` | 2 | 全部支持 | 全部支持 |
| 动画状态 | `AnimationState` | 6 | 全部支持 | 全部支持 |
| 轨道条目 | `TrackEntry` | 6 | 全部支持 | 全部支持 |

## 更新阶段回调（仅 SkeletonAnimation）

在 `SkeletonAnimation.Update()` 中依次触发：

| 回调 | 时机 | 用途 | 可写 |
|:-----|:-----|:-----|:-----|
| `BeforeApply` | 动画应用到骨骼前 | 设置动画不驱动的骨骼兜底值 | 会 |
| `UpdateLocal` | 局部值写入后，世界值未算 | 读取/修改局部值 | 会 |
| `UpdateComplete` | 世界值计算后 | 只读世界值（挂特效/碰撞） | **否** |
| `UpdateWorld` | 世界值计算后 | 修改局部值→触发第二次重算 | 会 |

> [!warning] `UpdateWorld` 性能代价
> 订阅此事件会触发**第二次 `skeleton.UpdateWorldTransform()`**。复杂 skeleton 上此开销不可忽视，仅在需要自定义约束时使用。

## 渲染阶段回调（SA + SG）

| 回调 | 频率 | 时机 | 用途 |
|:-----|:-----|:-----|:-----|
| `OnRebuild` | 一次性 | Skeleton 初始化后 | 添加自定义材质、缓存引用 |
| `OnMeshAndMaterialsUpdated` | **每帧** | `LateUpdate` 末尾 | 动态修改材质属性、后处理渲染 |

`OnMeshAndMaterialsUpdated` 是最常用的渲染 delegate——每帧 mesh 和 Material 更新后的"最后一站"。

> [!warning] 订阅管理
> 此 delegate 每帧触发，GameObject 销毁后必须 `-=` 取消订阅，否则导致内存泄露。见 [[Spine资源管理]] 的 Dispose 流程。

## 动画状态事件（AnimationState）

六种全局事件（所有轨道）：

| 事件 | 触发条件 |
|:-----|:-----|
| `Start` | 动画开始播放 |
| `Interrupt` | 被新动画中断（不触发 Complete） |
| `Complete` | 完整播放一次（循环动画每循环触发） |
| `End` | 动画播放结束 |
| `Dispose` | TrackEntry 被销毁 |
| `Event` | Spine 编辑器自定义事件 |

**触发顺序**：TrackEntry 事件先于 AnimationState 同名事件。

**性能关键**：缓存 `Spine.EventData` 引用，`e.Data == cachedEventData` 引用比较，零 GC。

## 轨道条目事件（TrackEntry）

`SetAnimation` / `AddAnimation` 返回的 `TrackEntry` 提供同名六种事件，仅对该次播放生效。

**选择指南**：
- 全局逻辑（状态机、UI 更新）→ AnimationState 全局事件
- 精确到某次播放（技能回调）→ TrackEntry 单实例事件

> [!warning] TrackEntry 生命周期
> `Dispose` 事件触发后 TrackEntry 失效。动画切换或 `ClearTrack` 触发 Dispose。

## SkeletonGraphic vs SkeletonAnimation

`SkeletonGraphic` 继承自 `MaskableGraphic`（Unity UI 体系），使用 `CanvasRenderer` 渲染。**不暴露**四个更新阶段回调。

| 特性 | SkeletonAnimation | SkeletonGraphic |
|:-----|:-----|:-----|
| 渲染器 | `MeshRenderer` | `CanvasRenderer` |
| 更新回调 | BeforeApply 等 4 个 | 无 |
| OnRebuild / OnMeshAndMaterialsUpdated | 支持 | 支持 |
| AnimationState 事件 | 支持 | 支持 |
| 推荐场景 | 游戏角色/物体 | UI Spine 立绘 |

## 常见陷阱

1. **未取消订阅** — `OnMeshAndMaterialsUpdated` 每帧触发，Destroy 前不 `-=` 导致目标对象无法 GC
2. **`UpdateComplete` 中修改值** — 当前帧不生效，需用 `UpdateWorld`
3. **TrackEntry Dispose 后使用** — 动画被移除后引用失效
4. **SG 上使用更新回调** — `SkeletonGraphic` 无 `BeforeApply` 等四个回调
5. **回调中 GC** — 每帧 `GetComponent` / `FindBone` / `new MaterialPropertyBlock` / `.material` 应缓存或复用
6. **`UpdateWorld` 隐形成本** — 第二次 `UpdateWorldTransform` 的 CPU 开销

## 参见

- [[sources/spine-delegates-摘要|来源摘要]]
- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — 基础概念
- [[concepts/Spine资源管理|Spine 资源管理]] — Dispose 中的 delegate 清理
