---
title: "Unity 动画与 Spine"
type: concept
updated: 2026-05-11
tags: [unity, animation, spine, animator]
---

# Unity 动画与 Spine

Unity 动画系统以 Animator Controller + Animation Clip + Avatar 为现代工作流核心，通过路径匹配和骨骼映射实现动画复用；Spine 是 2D 骨骼动画工具，通过 `spine-unity` 运行时包集成，提供 Skeleton / SkeletonAnimation / AnimationState 组件及事件驱动架构。

## Unity 动画基础

### Animation Clip

Animation Clip 是 Unity 中最基础的动画数据单元，可控制 GameObject 上任意组件的属性变换（Transform、材质颜色、UI 元素等）。动画曲线支持 Constant、Linear、Freeform（Bezier）插值模式。

**路径匹配（Path Matching）** 是动画复用的核心机制：Animation Clip 通过子物体的相对路径匹配目标对象的层级结构，只要名称和层级相同即可复用。这正是 Humanoid 动画跨模型复用的底层原理。

### Root Motion

控制动画是否驱动 GameObject 的世界空间位移。

| 模式 | 行为 |
|:-----|:-----|
| 关闭 | 动画直接覆盖世界坐标（绝对值），角色位置完全由动画曲线决定 |
| 开启 | 使用 deltaPosition/deltaRotation 增量插值，配合物理和脚本实现自然移动 |

`Bake Into Pose` 决定根骨骼变换是作为 Root Motion 驱动 GameObject，还是作为普通动画烘焙到骨骼姿态中。通过覆写 `OnAnimatorMove()` 可以在代码层面完全控制 Root Motion。

### Avatar 与动画复用

Avatar 是 Unity 的骨骼映射系统，将模型骨骼映射为标准化的人形骨骼（Muscle），使不同骨骼结构的模型共享同一套动画。创建流程：选中模型 → Rig 选项卡 → 设置 Animation Type → Create From This Model → Configure 骨骼映射。必选骨骼用实线表示，可选骨骼用虚线表示。

## Animator 状态机

### Animator 组件

Animator 是根控制器组件，核心属性：

| 属性 | 说明 |
|:-----|:-----|
| Apply Root Motion | 是否使用动画位移驱动 GameObject |
| Update Mode | Normal（Update）/ Animate Physics（FixedUpdate）/ Unscaled Time（不受 timeScale 影响） |
| Culling Mode | Always Animate / Cull Update Transforms（跳过 Transform 更新）/ Cull Completely（完全停止） |

### Layer 与 Parameters

Layer 通过权重和 Avatar Mask 组合不同身体部位的动画。Parameters 提供四种类型：**Float**（连续值）、**Int**（离散状态）、**Bool**（开关）、**Trigger**（瞬时触发，自动复位）。推荐用 `Animator.StringToHash()` 缓存参数 Hash 值，避免字符串 GC。

状态属性：Speed（播放速度）、Multiplier（参数绑定速度乘数）、Motion Time（归一化时间点）、Mirror（仅 Humanoid 有效）、Cycle Offset（循环起始偏移）、Foot IK（脚部 IK 绑定）、Write Defaults（对象池场景控制）。

### 动画转换（Transitions）

状态间转换由 Conditions 触发，Has Exit Time 控制是否等待当前动画完成到指定阈值。Interruption Source 决定转换打断行为（Current/Next/Current Then Next/Next Then Current）。Solo/Mute 控制转换优先级——Solo 独占当前状态，Mute 永久禁用。

### BlendTree

通过参数控制多动画平滑混合：

| 类型 | 用途 |
|:-----|:-----|
| 1D | 单参数线性混合（Idle→Walk→Run） |
| 2D Simple Directional | 方向控制（前后左右） |
| 2D Freeform Directional | 自由方向混合 |
| 2D Freeform Cartesian | 两个独立参数 |
| Direct | 直接控制每个动画权重 |

BlendTree 可嵌套子 BlendTree，实现速度 + 方向 + 坡度等多维混合。

## Spine 骨骼动画

### Skeleton 组件

Skeleton 是 Spine 的数据载体，存储骨骼层级、Slot、Attachment 和 Skin。`SetAttachment(slotName, attachmentName)` 切换槽位附件，`SetSkin(skinName)` 换肤。支持组合皮肤（Mix-and-Match）：创建 Skin 实例后 `AddSkin()` 叠加多个预设皮肤，配合 `GetRepackedSkin()` 将多皮肤纹理合并为单张图集以减少 Draw Call。

### SkeletonAnimation 生命周期

四个更新回调阶段（详见 `[[Spine Delegates]]`）：

| 回调 | 时机 | 用途 |
|:-----|:-----|:-----|
| BeforeApply | 动画应用到 Skeleton 前 | 修改骨骼状态 |
| UpdateLocal | 动画更新完成、局部值写入后 | 读取或修改局部值 |
| UpdateComplete | 所有骨骼世界值计算后 | 只读世界值 |
| UpdateWorld | 世界值计算后 | 根据世界值修改局部值（自定义约束） |

`Update(deltaTime)` 完整推进动画，`Update(0)` 不推进时间仅刷新姿态。`[DefaultExecutionOrder(-1)]` 可在 SkeletonAnimation 之前执行自定义脚本。`SkeletonGraphic` 不暴露这四个回调。


### AnimationState 控制

多 Track 动画管理：

```csharp
// 播放（推荐 AnimationReferenceAsset 避免字符串查找）
TrackEntry entry = anim.AnimationState.SetAnimation(0, walkAnim, true);
// 队列播放
entry = anim.AnimationState.AddAnimation(0, runAnim, true, 2f);
// 空动画淡出
entry = anim.AnimationState.SetEmptyAnimation(0, 0.3f);
// 清空轨道
anim.AnimationState.ClearTrack(0);
anim.AnimationState.ClearTracks();
```

返回的 `TrackEntry` 支持 TimeScale、MixDuration、Alpha 等参数定制。TrackEntry 在动画移除后失效并被 GC 回收。

### 事件系统

六种事件类型：**Start** / **Interrupt** / **Complete** / **End** / **Dispose** / **Event**（用户自定义）。TrackEntry 单实例事件先于 AnimationState 全局事件触发。性能关键：缓存 `Spine.EventData` 引用，使用 `e.Data == cachedEventData` 引用比较，零 GC 开销。

### Material 与换装

四种换装策略：

| 策略 | 复杂度 | 性能 | 灵活性 |
|:-----|:-----|:-----|:-----|
| Skin 换皮 | 低 | 高 | 低（需美术预定义） |
| 手动 Attachment | 中 | 低 | 高 |
| CustomTextureOverride | 中 | 中 | 中（需同尺寸纹理） |
| MaterialPropertyBlocks | 高 | 最高 | 中（不支持 SkeletonGraphic） |

> MaterialPropertyBlocks 的不同参数值会破坏合批（Batching）。缓存 `Shader.PropertyToID()` 结果和 MaterialPropertyBlock 实例以优化性能。

## 参见

- [[sources/Unity动画与Spine-摘要|来源摘要]]
- [[Spine Delegates]] — delegate/event 系统深度解析
- [[Spine资源管理]] — 资源卸载与 Dispose 流程
