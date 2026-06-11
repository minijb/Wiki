---
title: "Spine 资源卸载指南 — 摘要"
type: source-summary
updated: 2026-06-11
source: "raw/gamedev/animation/spine-resource-unload.md"
tags: [spine, unity, resource-management, memory, dispose]
---

# Spine 资源卸载指南

## 来源

`raw/gamedev/animation/spine-resource-unload.md` — 基于真实项目的 Spine 资源卸载实战指南，覆盖引用链分析、三步清理法、ResourceManager 交互和 Lua 层注意事项

## 要点

1. **资源引用链** — `SkeletonDataAsset → AtlasAsset → Material → Texture2D` 多层依赖；同时存在 AssetBundle 加载的磁盘资产和 `CanvasRenderer` 动态分配的材质槽
2. **清理三步法（顺序不可调换）** — 步骤 1：在 `skeletonDataAsset = null` 之前，用 `Resources.UnloadAsset()` 卸载 atlas 纹理和 Material；步骤 2：`skeletonGraphic.Clear()` + 断开 `skeletonDataAsset` 引用；步骤 3：遍历 `canvasRenderer.materialCount` 逐槽清除
3. **活跃 vs 缓存池** — 必须同时处理当前活跃的 `_skeletonDataAsset` 和 `_recycleDataAssets` 缓存池；活跃的必须在 `skeletonDataAsset = null` 之前处理
4. **多渲染器模式** — 启用 `allowMultipleCanvasRenderers` 时，需额外遍历 `skeletonGraphic.canvasRenderers` 列表清理子渲染器材质槽
5. **常见陷阱** — `DestroyImmediate` 对 AssetBundle 资产抛异常（运行时禁用）；`SetMaterial(null, 0)` 在 `materialCount == 0` 时索引越界；`_defaultMaterial` 字段（`_SetGray()` 加载）持续持有引用；仅设 `skeletonGraphic.material = null` 不清除多槽位
6. **ResourceManager 交互** — `AssetBundle.UnloadAsync(false)` 保留已加载对象（纹理不释放），需配合 `Resources.UnloadAsset` 即时释放；Editor 模式下 `Resources.UnloadAsset` 被注释导致资源永不释放；卸载有 15-20 秒双重延迟
7. **Lua 层协调** — 面板销毁时 `OnDestroy → Dispose` 链自动触发；面板缓存（仅 Hide）时需主动调用 `Dispose()`，下次激活重新 `Initialize`
8. **验证方法** — Profiler Memory 面板搜索纹理/Material、`Resources.FindObjectsOfTypeAll<>()` 对比、Memory Profiler 包快照对比

## 关联 Wiki 页面

- [[concepts/Spine资源管理|Spine 资源管理]] — 概念页
- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — 基础概念
- [[concepts/Spine-Delegates|Spine Delegates]] — delegate 清理与生命周期管理
