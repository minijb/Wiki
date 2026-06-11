---
status: refined
merged-to: drafts/Spine资源卸载指南.md
updated: 2026-06-11
---

# Spine 资源泄露 — 迭代修复记录

## 背景

`SpineUIController.Dispose()` 原有逻辑仅取消事件订阅和调用 ResourceManager 异步卸载，未清理 `SkeletonGraphic` 的内部状态和 `SkeletonDataAsset` 的依赖引用链，导致 GameObject 销毁后 Material 和 Texture2D 残留。

以下记录五次迭代的完整过程：每次的代码改动、验证结果和失败原因。

## 目标文件

`Client/Assets/Script/Core/Graphic/SpineUIController.cs` — `Dispose()` 方法

---

### 第 1 版 — 初始修复

**改动**：在 `Dispose()` 中新增 SkeletonGraphic 清理 + 断开 AtlasAsset 引用链

```csharp
public void Dispose()
{
    if (skeletonGraphic != null)
    {
        skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;

        // 新增：清理 SkeletonGraphic 内部状态
        skeletonGraphic.skeletonDataAsset = null;
        skeletonGraphic.Clear();
        skeletonGraphic.material = null;
        if (skeletonGraphic.canvasRenderer != null)
        {
            skeletonGraphic.canvasRenderer.SetMaterial(null, 0);
        }
    }

    _skeletonDataAsset = null;
    _skeletonData = null;

    foreach (var item in _recycleDataAssets)
    {
        // 新增：断开 SkeletonDataAsset → AtlasAsset 引用链
        if (item.Value != null && item.Value.atlasAssets != null)
        {
            item.Value.atlasAssets = new AtlasAssetBase[0];
        }

        ResourceManager.Inst.UnloadAsync(item.Key);
        DestroyImmediate(item.Value);
    }
    // ... _recycleModels 循环不变
}
```

**验证结果**：运行时抛异常 `Failed setting material. Index is out of bounds.`

**原因**：`canvasRenderer.SetMaterial(null, 0)` 在 `materialCount == 0` 时索引越界。

---

### 第 2 版 — 修复 IndexOutOfBounds

**改动**：移除 `canvasRenderer.SetMaterial(null, 0)` 调用

```diff
-            if (skeletonGraphic.canvasRenderer != null)
-            {
-                skeletonGraphic.canvasRenderer.SetMaterial(null, 0);
-            }
```

**验证结果**：不再抛异常，但测试发现 SkeletonDataAsset 已清理，**Material 和 Texture2D 仍然残留**。

**原因**：仅设 `skeletonGraphic.material = null` 只清除了 Graphic 基类的默认材质引用。`SkeletonGraphic` 为多页 atlas 创建了多个材质槽写入 CanvasRenderer，这些槽位未被清理，Material → Texture2D 引用链仍完整。

---

### 第 3 版 — 增加 CanvasRenderer 逐槽清理

**改动**：遍历 `materialCount`，逐槽清除材质并销毁

```csharp
if (skeletonGraphic.canvasRenderer != null)
{
    int materialCount = skeletonGraphic.canvasRenderer.materialCount;
    for (int i = 0; i < materialCount; i++)
    {
        Material mat = skeletonGraphic.canvasRenderer.GetMaterial(i);
        skeletonGraphic.canvasRenderer.SetMaterial(null, i);
        if (mat != null)
        {
            DestroyImmediate(mat);
        }
    }
}
```

**验证结果**：运行时抛异常 `Destroying assets is not permitted to avoid data loss.`

**原因**：原代码 `DestroyImmediate(item.Value)`（SkeletonDataAsset）在 Editor 外运行时会抛此异常。Unity 不允许在运行时对 Asset 类型对象调用 `DestroyImmediate`。同时，CanvasRenderer 中新增的 `DestroyImmediate(mat)` 也可能命中此问题 — Material 可能是 AssetBundle 加载的资产引用而非运行时实例。

---

### 第 4 版 — 移除 DestroyImmediate(asset) + 断开引用替代

**改动**：
- 移除 `_recycleDataAssets` 循环中的 `DestroyImmediate(item.Value)`，由 ResourceManager 全权负责卸载
- CanvasRenderer 清理改为 `mat.mainTexture = null` 而非 `DestroyImmediate(mat)`

```csharp
// _recycleDataAssets 循环
foreach (var item in _recycleDataAssets)
{
    if (item.Value != null && item.Value.atlasAssets != null)
    {
        item.Value.atlasAssets = null;
    }
    ResourceManager.Inst.UnloadAsync(item.Key);
    // DestroyImmediate 已移除
}

// CanvasRenderer 清理
if (skeletonGraphic.canvasRenderer != null)
{
    int materialCount = skeletonGraphic.canvasRenderer.materialCount;
    for (int i = 0; i < materialCount; i++)
    {
        Material mat = skeletonGraphic.canvasRenderer.GetMaterial(i);
        skeletonGraphic.canvasRenderer.SetMaterial(null, i);
        if (mat != null)
        {
            mat.mainTexture = null;  // 断开引用，不销毁 Material
        }
    }
}
```

**验证结果**：缓存的 `_recycleDataAssets` 已清理，但**当前活跃的 `_skeletonDataAsset` 的纹理仍然存在**。

**原因**：`_recycleDataAssets` 中的 SkeletonDataAsset 通过 `atlasAssets = null` + `UnloadAsync` 被正确处理。但当前正在显示的 SkeletonDataAsset（`_skeletonDataAsset` 字段）在 `skeletonGraphic.skeletonDataAsset = null` 之后就被断开了关联，它的 atlas 纹理从未被显式释放 — 代码只处理了缓存池中的，没处理活跃的那个。

---

### 第 5 版 — 显式卸载活跃 SkeletonDataAsset 的纹理

**改动**：在清理 SkeletonGraphic **之前**，先遍历 `_skeletonDataAsset.atlasAssets[].Materials[].mainTexture`，调用 `Resources.UnloadAsset(tex)` 显式卸载纹理。

```csharp
if (skeletonGraphic != null)
{
    skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;

    // 关键：必须在 skeletonDataAsset = null 之前访问 _skeletonDataAsset
    if (_skeletonDataAsset != null && _skeletonDataAsset.atlasAssets != null)
    {
        foreach (var atlas in _skeletonDataAsset.atlasAssets)
        {
            if (atlas != null && atlas.Materials != null)
            {
                foreach (var mat in atlas.Materials)
                {
                    if (mat != null && mat.mainTexture != null)
                    {
                        Texture tex = mat.mainTexture;
                        mat.mainTexture = null;
                        Resources.UnloadAsset(tex);
                    }
                }
            }
        }
    }

    skeletonGraphic.skeletonDataAsset = null;
    skeletonGraphic.Clear();
    skeletonGraphic.material = null;

    // CanvasRenderer 逐槽清理（同第 4 版）
    if (skeletonGraphic.canvasRenderer != null)
    {
        int materialCount = skeletonGraphic.canvasRenderer.materialCount;
        for (int i = 0; i < materialCount; i++)
        {
            Material mat = skeletonGraphic.canvasRenderer.GetMaterial(i);
            skeletonGraphic.canvasRenderer.SetMaterial(null, i);
            if (mat != null)
            {
                mat.mainTexture = null;
            }
        }
    }
}
```

**验证结果**：纹理已被清理，但**还有 2 个 Material 对象残留**。

**原因**：
- Atlas 的 Material 对象本身未被卸载——v5 只调用了 `Resources.UnloadAsset(tex)` 释放纹理，Material 资产仍然存在
- `_defaultMaterial` 字段（在 `_SetGray()` 中加载，`Dispose()` 从未清理）持续持有对 spine UI 材质的引用

---

### 第 6 版（最终） — 卸载 Material 并清理 _defaultMaterial

**改动**：
- Atlas 循环中追加 `Resources.UnloadAsset(mat)` 卸载 Material 自身
- 新增 `_defaultMaterial = null` 释放 `_SetGray()` 加载的 UI spine 材质引用

```csharp
// atlas 清理循环（新增 mat 卸载）
foreach (var mat in atlas.Materials)
{
    if (mat != null)
    {
        if (mat.mainTexture != null)
        {
            Texture tex = mat.mainTexture;
            mat.mainTexture = null;
            Resources.UnloadAsset(tex);
        }
        Resources.UnloadAsset(mat);  // 新增：卸载 Material 自身
    }
}

// 新增：释放 _defaultMaterial 引用
_skeletonDataAsset = null;
_skeletonData = null;
_defaultMaterial = null;
```

**验证结果**：纹理和 Material 均被清理；`_recycleDataAssets` 缓存正常；活跃 SkeletonDataAsset 的 atlas 资源完全释放。

---

## 迭代总结

| 版本 | 解决的问题 | 引入/暴露的问题 |
|------|-----------|----------------|
| v1 | 首次加入 SkeletonGraphic 清理 + AtlasAsset 断开 | `SetMaterial(null, 0)` IndexOutOfBounds |
| v2 | 移除越界的 SetMaterial 调用 | Material/Texture 仍残留（CanvasRenderer 多槽位未清理） |
| v3 | 逐槽遍历 CanvasRenderer + DestroyImmediate | "Destroying assets is not permitted" 运行时异常 |
| v4 | 移除所有 asset DestroyImmediate；CanvasRenderer 改用 mainTexture=null | 缓存清理正常，但活跃 _skeletonDataAsset 纹理未处理 |
| v5 | 在 skeletonDataAsset=null 之前显式 `Resources.UnloadAsset(tex)` | 纹理已清理，但 2 个 Material 仍残留 |
| v6 | 追加 `Resources.UnloadAsset(mat)` + `_defaultMaterial = null` | — |
## 最终版完整代码

```csharp
public void Dispose()
{
    if (skeletonGraphic != null)
    {
        skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;

        // 步骤1：卸载当前 SkeletonDataAsset 的 atlas 纹理和 Material
        // 顺序关键 — 必须在 skeletonGraphic.skeletonDataAsset = null 之前访问 _skeletonDataAsset
        if (_skeletonDataAsset != null && _skeletonDataAsset.atlasAssets != null)
        {
            foreach (var atlas in _skeletonDataAsset.atlasAssets)
            {
                if (atlas != null && atlas.Materials != null)
                {
                    foreach (var mat in atlas.Materials)
                    {
                        if (mat != null)
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
                }
            }
        }

        // 步骤2：清理 SkeletonGraphic 内部状态
        skeletonGraphic.skeletonDataAsset = null;
        skeletonGraphic.Clear();
        skeletonGraphic.material = null;

        // 步骤3：清除 CanvasRenderer 上的所有材质槽
        if (skeletonGraphic.canvasRenderer != null)
        {
            int materialCount = skeletonGraphic.canvasRenderer.materialCount;
            for (int i = 0; i < materialCount; i++)
            {
                Material mat = skeletonGraphic.canvasRenderer.GetMaterial(i);
                skeletonGraphic.canvasRenderer.SetMaterial(null, i);
                if (mat != null)
                {
                    mat.mainTexture = null;
                }
            }
        }
    }

    _skeletonDataAsset = null;
    _skeletonData = null;
    _defaultMaterial = null;

    foreach (var item in _recycleDataAssets)
    {
        if (item.Value != null && item.Value.atlasAssets != null)
        {
            item.Value.atlasAssets = null;
        }

        ResourceManager.Inst.UnloadAsync(item.Key);
    }
    _recycleDataAssets.Clear();
    foreach (var item in _recycleModels)
    {
        ResourceManager.Inst.UnloadAsync(item.Key);
        Destroy(item.Value);
    }
    _recycleModels.Clear();
    _isDirty = false;
}
```
