---
title: "Unity 动画系统与 Spine — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/animation/unity-animation-spine.md"
tags: [unity, animation, spine, animator]
---

# Unity 动画系统与 Spine

## 来源

`raw/gamedev/animation/unity-animation-spine.md` — Unity 动画系统（Animation Clip、Root Motion、Avatar、Animator Controller）与 Spine 2D 骨骼动画运行时集成

## 要点

1. **Animation Clip** — Unity 动画基础数据单元，控制任意组件的属性变换；通过路径匹配（Path Matching）实现复用——子物体名称和层级路径相同时动画可跨对象共用，这正是 Humanoid 动画复用的底层原理
2. **Root Motion** — 控制动画是否驱动 GameObject 世界空间位移；开启时使用 deltaPosition/deltaRotation 增量插值，关闭时动画数据直接覆盖世界坐标；`Bake Into Pose` 决定根骨骼变换是作为 Root Motion 还是烘焙到骨骼姿态中
3. **Avatar** — 骨骼映射系统，建立骨骼到 Muscle 的标准化映射，使不同骨骼结构的 Humanoid 模型共享同一套动画；Avatar 通过肌肉拉伸变换（Muscle-based Stretching）驱动不同体型
4. **Animator 组件** — 根控制器组件，提供 Update Mode（Normal/Animate Physics/Unscaled Time）、Culling Mode（Always Animate/Cull Update Transforms/Cull Completely）、Apply Root Motion 等核心属性
5. **Layer 与 Parameters** — Layer 通过权重和 Avatar Mask 组合不同身体部位动画；Parameters 支持 Float/Int/Bool/Trigger 四种类型，配合 `Animator.StringToHash()` 避免字符串 GC 开销
6. **动画状态与转换** — 三种状态类型：单 Clip / BlendTree / 子状态机；Transitions 通过 Has Exit Time、Interruption Source、Solo/Mute 优先级控制切换行为；`StateInfo.IsTag()` 运行时判断状态
7. **BlendTree** — 通过参数控制多动画混合：1D（速度线性混合）、2D Simple Directional / Freeform Directional / Freeform Cartesian（方向+速度）、Direct（直接控制权重）；BlendTree 可嵌套实现多维混合
8. **Spine Skeleton** — 骨骼数据载体，存储骨骼层级、Slot、Attachment 和 Skin；支持组合皮肤（Mix-and-Match）和 `GetRepackedSkin()` 重打包以减少 Draw Call
9. **SkeletonAnimation 生命周期** — 四个更新阶段：BeforeApply → UpdateLocal → UpdateComplete → UpdateWorld；`Update(deltaTime)` 完整更新，`Update(0)` 不推进时间刷新姿态
10. **AnimationState** — 多 Track 播放管理：`SetAnimation` 播放、`AddAnimation` 队列、`SetEmptyAnimation`/`AddEmptyAnimation` 淡入淡出；返回 `TrackEntry` 支持 TimeScale/MixDuration/Alpha 等定制；TrackEntry 在动画移除后失效
11. **事件系统** — Start / Interrupt / Complete / End / Dispose / Event 六种事件；TrackEntry 单实例事件先于 AnimationState 全局事件触发；引用比较 `e.Data == cachedEventData` 零 GC 开销
12. **Material 与换装** — 四种策略：Skin 换皮（简单但需预定义）、手动 Attachment（万能但性能开销大）、CustomTextureOverride（需同尺寸纹理）、MaterialPropertyBlocks（性能最优但不支持 SkeletonGraphic）；不同 PropertyBlock 参数值会破坏合批

## 关联 Wiki 页面

- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — 概念页
