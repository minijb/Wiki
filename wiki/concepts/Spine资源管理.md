---
title: "Spine 资源管理"
type: concept
updated: 2026-06-11
tags: [spine, unity, resource-management, memory, dispose]
---

# Spine 资源管理

Spine 资源（`SkeletonDataAsset`、`AtlasAsset`、`Material`、`Texture2D`）形成多层引用链。正确卸载需要理解这条链，并按特定顺序断开每一层引用。

## 资源引用链

```
SkeletonDataAsset (ScriptableObject)
  └── atlasAssets: AtlasAsset[] (ScriptableObject)
       └── Materials[]: Material[]          ← AssetBundle 加载的磁盘资产
            └── mainTexture: Texture2D      ← GPU 原生像素内存

SkeletonGraphic
  ├── skeletonDataAsset  ─────────────→ SkeletonDataAsset
  ├── canvasRenderer
  │    └── materialCount 个槽位         ← 每个 atlas page 一个槽
  │         └── GetMaterial(i) ───────→ 引用 atlas Material
  └── canvasRenderers[]                 ← 多渲染器模式下的子渲染器
```

## 清理三步法

**顺序不可调换。**

### 步骤 1：卸载 atlas 纹理和 Material

在 `skeletonGraphic.skeletonDataAsset = null` **之前**执行。Material 和 Texture2D 来自 AssetBundle，用 `Resources.UnloadAsset()` 立即释放原生内存：

```csharp
foreach (var atlas in _skeletonDataAsset.atlasAssets)
{
    foreach (var mat in atlas.Materials)
    {
        if (mat.mainTexture != null)
        {
            Texture tex = mat.mainTexture;
            mat.mainTexture = null;
            Resources.UnloadAsset(tex);
        }
        Resources.UnloadAsset(mat);
    }
}
```

> [!warning] `Resources.UnloadAsset` 仅对磁盘 Asset 有效
> 运行时 `new Material(shader)` 动态创建的材料不受此方法影响，应用 `Destroy`。

### 步骤 2：清理 SkeletonGraphic 内部状态

```csharp
skeletonGraphic.skeletonDataAsset = null;
skeletonGraphic.Clear();        // 释放内部 Mesh/骨骼/材质列表
skeletonGraphic.material = null; // 清除 Graphic 基类的默认材质
```

### 步骤 3：清除 CanvasRenderer 材质槽

`SkeletonGraphic` 为多页 atlas 创建的额外材质槽仅设 `material = null` 不会清除：

```csharp
for (int i = 0; i < skeletonGraphic.canvasRenderer.materialCount; i++)
{
    Material mat = skeletonGraphic.canvasRenderer.GetMaterial(i);
    skeletonGraphic.canvasRenderer.SetMaterial(null, i);
    if (mat != null) mat.mainTexture = null; // 断开引用，不销毁
}
```

多渲染器模式下需额外遍历 `canvasRenderers` 列表。

## 常见陷阱

### 1. `DestroyImmediate` 对资产抛异常

Unity 不允许运行时对 Asset 类型调用 `DestroyImmediate`。解决方案：断开引用链 + 由 `Resources.UnloadAsset` 释放。

### 2. 活跃 vs 缓存池 SkeletonDataAsset

只清理了 `_recycleDataAssets`（换装缓存），漏掉当前活跃的 `_skeletonDataAsset`。两者都需处理，且活跃的必须在 `skeletonDataAsset = null` 之前。

### 3. Material 残留

仅释放纹理（`Resources.UnloadAsset(tex)`），Material 对象仍占用内存。需追加 `Resources.UnloadAsset(mat)`。

### 4. `_defaultMaterial` 持有额外引用

灰度化功能加载的材质持续引用，Dispose 中必须 `_defaultMaterial = null`。

### 5. CanvasRenderer 多槽遗漏

仅设 `skeletonGraphic.material = null` 只清除 Graphic 基类的默认槽（索引 0），多页 atlas 的额外槽位（索引 1+）仍残留。

### 6. `OnMeshAndMaterialsUpdated` 未取消订阅

GameObject 销毁前必须取消 `OnMeshAndMaterialsUpdated` 委托订阅，否则形成 `SkeletonRenderer → delegate → 目标对象` 的强引用链，阻止 GC。

## ResourceManager 交互

- `AssetBundle.UnloadAsync(false)` 保留已加载对象，纹理在 Bundle 卸载后仍存在
- 卸载有两段延迟（AssetProxy ~10s + BundleProxy ~5-10s）
- Editor 模式 `Resources.UnloadAsset` 常被注释，资源永不释放
- `Dispose()` 中的直接 `Resources.UnloadAsset` 提供了即时释放通道

## Lua 层协调

| 场景 | 清理行为 |
|:-----|:-----|
| 面板销毁（Destroy → OnDestroy → Dispose） | 自动触发完整清理 |
| 面板缓存（仅 Hide） | **需主动调 Dispose()**，下次打开重新 Initialize |

## 为什么必须主动释放（实战印证）

清理三步法不是"可选优化"，而是**避免泄漏的必要操作**——以下机制决定了 GC 不可靠，必须在 `OnDestroy`/`Dispose` 主动断开所有资源引用。

### GameObject 销毁 ≠ 托管对象释放

`Destroy(go)` 只销毁原生对象，C# 托管包装（`SpineUIController`、`SkeletonGraphic`）变 fake null 但仍存活，字段引用依旧有效。只要 .NET GC 没跑，被引用的资源就不会被回收。因此释放点必须落在必然执行的 `OnDestroy`/`Dispose`。

### xLua ObjectTranslator 钉住

被 Lua 访问过的 C# 对象被 xLua 桥以强引用钉住，短期不可被 GC 回收。但这只是**次要因素**——只要释放点断开了资源引用（置空 `skeletonGraphic`、清 CanvasRenderer 槽、卸载 SkeletonData），被钉住的小对象即便存活也碰不到纹理。

### 真实案例

[[sources/spine-texture-leak-gongsunyue-摘要|paintinganim_gongsunyue 泄漏案例]]中，旧版 `OnDestroy` 漏清 CanvasRenderer 槽 + 未 `UnloadAsync` + 未置空 `skeletonGraphic`，导致 16 MB 纹理在 GO 销毁后仍驻留。补全三步即消除。完整的根因定位与辨析方法见 [[concepts/内存泄漏调试方法论|内存泄漏调试方法论]]。

## 参见

- [[sources/spine-resource-unload-摘要|来源摘要]]
- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — Spine 基础组件
- [[concepts/Spine-Delegates|Spine Delegates]] — delegate 生命周期与清理
