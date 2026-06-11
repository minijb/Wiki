---
title: "Spine Delegates 完全解析 — 摘要"
type: source-summary
updated: 2026-06-11
source: "raw/gamedev/animation/spine-delegates.md"
tags: [spine, unity, delegate, callback]
---

# Spine Delegates 完全解析

## 来源

`raw/gamedev/animation/spine-delegates.md` — Spine Unity 运行时 delegate/event 系统全解析，覆盖四组 delegate 的触发时机、参数、使用场景和常见陷阱

## 要点

1. **四组 delegate 分类** — 更新阶段回调（`SkeletonAnimation`，4 个）、渲染阶段回调（`SkeletonRenderer`，2 个）、动画状态事件（`AnimationState`，6 个）、轨道条目事件（`TrackEntry`，6 个）
2. **更新阶段回调** — `BeforeApply`（动画应用前）→ `UpdateLocal`（局部值写入后）→ `UpdateComplete`（世界值只读）→ `UpdateWorld`（修改局部值，触发第二次 `UpdateWorldTransform`，有性能代价）；仅 `SkeletonAnimation` 支持，`SkeletonGraphic` 无
3. **渲染阶段回调** — `OnRebuild`（Skeleton 初始化时一次性触发）和 `OnMeshAndMaterialsUpdated`（每帧 `LateUpdate` 结束触发）；SA 和 SG 均支持
4. **Unity 帧循环时序** — `Update` → 四个更新回调 → `LateUpdate` → `OnMeshAndMaterialsUpdated` → Canvas 管线；`SkeletonGraphic` 使用 `CanvasRenderer` 但回调时序一致
5. **AnimationState 事件** — Start / Interrupt / Complete / End / Dispose / Event 六种全局事件，TrackEntry 同名事件先于 AnimationState 触发；Interrupt 时不触发 Complete
6. **事件性能优化** — 缓存 `Spine.EventData` 引用，使用 `e.Data == cachedEventData` 引用比较实现零 GC
7. **SkeletonGraphic vs SkeletonAnimation** — SG 继承自 `MaskableGraphic`（非 `SkeletonRenderer`），无更新阶段回调，使用 `CanvasRenderer`；官方推荐 SA 用于游戏角色，SG 用于 UI
8. **常见陷阱** — 未取消订阅导致内存泄露、`UpdateComplete` 中修改值不生效、TrackEntry Dispose 后使用、SG 上使用 SA 专属回调、`OnMeshAndMaterialsUpdated` 中的 GC 陷阱（`.material` 每帧创建实例）、`UpdateWorld` 的第二次 UpdateWorldTransform 开销
9. **实战模式** — 订阅-取消订阅对（防泄露）、一次性初始化（OnRebuild）、骨骼世界坐标采集（UpdateComplete + 缓存 Bone 引用）

## 关联 Wiki 页面

- [[concepts/Spine-Delegates|Spine Delegates]] — 概念页
- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — 基础概念与 API 概览
- [[concepts/Spine资源管理|Spine 资源管理]] — Dispose 流程中的 delegate 清理
