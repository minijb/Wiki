---
title: "Spine 资源卸载指南"
type: source
updated: 2026-06-11
tags: [spine, unity, resource-management, memory, dispose]
---

# Spine 资源卸载指南

## 适用场景

- 项目使用自定义 ResourceManager（非 Unity 原生 Resources/Addressables）
- Lua 层通过 XLua 桥接 C# 组件
- `SpineUIController` 是项目自定义封装，包裹 `SkeletonGraphic`
- 同时存在 AssetBundle 模式和 Editor Resources 模式
- `SkeletonGraphic` 挂载在 UI Canvas 下，通过 `CanvasRenderer` 渲染

## 资源引用链

Spine 资源从加载到渲染，形成多层引用链。理解这条链是正确卸载的前提。

```
SkeletonDataAsset (ScriptableObject)
  └── atlasAssets: AtlasAsset[] (ScriptableObject)
       └── Materials[]: Material[]          ← AssetBundle 加载的磁盘资产
            └── mainTexture: Texture2D      ← 原生像素数据 (GPU 内存)

SpineUIController
  ├── _skeletonDataAsset  ──────────────→ 上述 SkeletonDataAsset
  ├── _defaultMaterial     ──────────────→ 通过 _SetGray() 加载的 UI spine 材质
  └── skeletonGraphic (SkeletonGraphic)
       ├── skeletonDataAsset  ───────────→ 同上 (C# 引用)
       ├── material (Graphic 基类) ───────→ 默认材质槽 (单槽)
       ├── canvasRenderer
       │    └── materialCount 个槽位       ← 每个 atlas page 对应一个槽
       │         └── GetMaterial(i) ──────→ 引用 atlas Material
       └── [多渲染器模式]
            └── canvasRenderers: List<CanvasRenderer>
                 └── 每个子 CanvasRenderer 持有 Material 引用
```

**关键点**：

| 引用类型 | 来源 | 释放方式 |
|---------|------|---------|
| SkeletonDataAsset / AtlasAsset | AssetBundle 加载 | ResourceManager 引用计数 + `UnloadAsync` |
| Material | AssetBundle 加载，被 AtlasAsset 引用 | `Resources.UnloadAsset(mat)` |
| Texture2D | AssetBundle 加载，被 Material.mainTexture 引用 | `Resources.UnloadAsset(tex)` |
| CanvasRenderer 材质槽 | `SkeletonGraphic.Initialize()` 分配 | 断开引用（不可 DestroyImmediate） |
| `_defaultMaterial` | `_SetGray()` 加载 | 置 null，由 GC/RM 回收 |

## 清理三步法

以下代码是 `SpineUIController.Dispose()` 的完整清理逻辑。三步**顺序不可调换**。

### 步骤 1：卸载 atlas 纹理和 Material

**必须在 `skeletonGraphic.skeletonDataAsset = null` 之前执行。** 一旦断开 `skeletonDataAsset` 引用，就失去了访问 atlas 纹理的入口。

Material 和 Texture2D 都是 AssetBundle 加载的**磁盘资产**——`Resources.UnloadAsset()` 对其有效，会立即释放原生内存。

```csharp
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
                    // 先释放纹理（纹理是独立的 GPU 原生资源）
                    if (mat.mainTexture != null)
                    {
                        Texture tex = mat.mainTexture;
                        mat.mainTexture = null;        // 断开 Material → Texture 引用
                        Resources.UnloadAsset(tex);    // 释放纹理原生内存
                    }
                    Resources.UnloadAsset(mat);         // 释放 Material 原生内存
                }
            }
        }
    }
}
```

> [!warning] `Resources.UnloadAsset` 只对磁盘 Asset 有效
> 此处 atlas Material 和 Texture2D 来自 AssetBundle → 属于磁盘 Asset → 调用有效。
> 若材料是运行时 `new Material(shader)` 动态创建的，则 `Resources.UnloadAsset` 是 no-op，应改用 `Destroy` 或 `DestroyImmediate(mat, allowDestroyingAssets: true)`（仅 Editor）。

### 步骤 2：清理 SkeletonGraphic 内部状态

`SkeletonGraphic.Clear()` 会释放内部 Mesh、骨骼、usedMaterials 列表等，但**不会**卸载 AssetBundle 层面的资产。此步骤断开 C# 层面的引用。

```csharp
skeletonGraphic.skeletonDataAsset = null;   // 断开 SkeletonDataAsset 引用
skeletonGraphic.Clear();                     // 释放内部 Mesh/骨骼/材质列表
skeletonGraphic.material = null;             // 清除 Graphic 基类的默认材质
```

### 步骤 3：清除 CanvasRenderer 上的所有材质槽

`SkeletonGraphic` 为多页 atlas 创建多个材质槽，写入 `CanvasRenderer`。仅设置 `skeletonGraphic.material = null`（步骤 2）只清除**基类**的默认材质，多槽不会被处理。

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
            mat.mainTexture = null;  // 断开引用，不销毁 Material 对象
        }
    }
}
```

> [!warning] 不要对 CanvasRenderer 上的 Material 调用 `DestroyImmediate`
> Material 可能是 AssetBundle 加载的**资产引用**（而非运行时实例副本），对资产调用 `DestroyImmediate` 在运行时抛 `"Destroying assets is not permitted"`。解决方案是断开引用链（`mainTexture = null`）而非销毁对象，由步骤 1 的 `Resources.UnloadAsset` 负责释放原生内存。

### 多渲染器模式（`allowMultipleCanvasRenderers = true`）

当启用多渲染器时，SkeletonGraphic 会将子网格分配到多个**子 CanvasRenderer**（存储在 `canvasRenderers` 列表中）。此时需要遍历清理：

```csharp
// 在主 canvasRenderer 清理之外，追加子渲染器清理
foreach (var cr in skeletonGraphic.canvasRenderers)
{
    if (cr == null) continue;
    int count = cr.materialCount;
    for (int i = 0; i < count; i++)
    {
        Material mat = cr.GetMaterial(i);
        cr.SetMaterial(null, i);
        if (mat != null) mat.mainTexture = null;
    }
}
```

> SkeletonGraphic `Clear()` 方法**已经遍历** `canvasRenderers` 并调用 `Clear()`，但 `CanvasRenderer.Clear()` 不保证清除手动设置的材质槽。额外手动清理是最安全的做法。

### 断开 SkeletonDataAsset 引用链

处理完活跃的 SkeletonDataAsset 后，还需处理缓存池中的：

```csharp
foreach (var item in _recycleDataAssets)
{
    if (item.Value != null && item.Value.atlasAssets != null)
    {
        item.Value.atlasAssets = null;   // 断开 SkeletonDataAsset → AtlasAsset 引用链
    }
    ResourceManager.Inst.UnloadAsync(item.Key);
}
_recycleDataAssets.Clear();
```

> [!tip] 不调用 `DestroyImmediate(item.Value)`
> v1–v3 调用 `DestroyImmediate(SkeletonDataAsset)`，在运行时抛异常。ResourceManager 的 `UnloadAsync` 已通过引用计数管理生命周期，无需额外销毁。

## 完整 Dispose() 代码

```csharp
public void Dispose()
{
    if (skeletonGraphic != null)
    {
        skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;

        // 步骤1：卸载 atlas 纹理和 Material（必须在 skeletonDataAsset = null 之前）
        if (_skeletonDataAsset != null && _skeletonDataAsset.atlasAssets != null)
        {
            foreach (var atlas in _skeletonDataAsset.atlasAssets)
            {
                if (atlas != null && atlas.Materials != null)
                {
                    foreach (var mat in atlas.Materials)
                    {
                        if (mat == null) continue;
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

        // 步骤2：清理 SkeletonGraphic 内部状态
        skeletonGraphic.skeletonDataAsset = null;
        skeletonGraphic.Clear();
        skeletonGraphic.material = null;

        // 步骤3：清除 CanvasRenderer 所有材质槽
        if (skeletonGraphic.canvasRenderer != null)
        {
            int materialCount = skeletonGraphic.canvasRenderer.materialCount;
            for (int i = 0; i < materialCount; i++)
            {
                Material mat = skeletonGraphic.canvasRenderer.GetMaterial(i);
                skeletonGraphic.canvasRenderer.SetMaterial(null, i);
                if (mat != null) mat.mainTexture = null;
            }
        }

        // 多渲染器模式下的子 CanvasRenderer 清理
        foreach (var cr in skeletonGraphic.canvasRenderers)
        {
            if (cr == null || cr == skeletonGraphic.canvasRenderer) continue;
            int count = cr.materialCount;
            for (int i = 0; i < count; i++)
            {
                Material mat = cr.GetMaterial(i);
                cr.SetMaterial(null, i);
                if (mat != null) mat.mainTexture = null;
            }
        }
    }

    // 清理控制器自身字段
    _skeletonDataAsset = null;
    _skeletonData = null;
    _defaultMaterial = null;  // _SetGray() 加载的 UI spine 材质

    // 清理缓存池中的 SkeletonDataAsset 引用链
    foreach (var item in _recycleDataAssets)
    {
        if (item.Value != null && item.Value.atlasAssets != null)
        {
            item.Value.atlasAssets = null;
        }
        ResourceManager.Inst.UnloadAsync(item.Key);
    }
    _recycleDataAssets.Clear();

    // 清理缓存池中的 GameObject 模型
    foreach (var item in _recycleModels)
    {
        ResourceManager.Inst.UnloadAsync(item.Key);
        Destroy(item.Value);
    }
    _recycleModels.Clear();

    _isDirty = false;
}
```

## 常见陷阱

### 1. `SetMaterial(null, 0)` 索引越界

`materialCount` 可能为 0。在遍历材质槽之前必须检查 `materialCount > 0`，或使用 `for (int i = 0; i < materialCount; i++)` 循环（`materialCount == 0` 时循环体不执行）。

### 2. `DestroyImmediate` 在运行时对资产抛异常

Unity 不允许在运行时对 Asset 类型对象调用 `DestroyImmediate`（不带 `allowDestroyingAssets: true`）。报错：`"Destroying assets is not permitted to avoid data loss."`

解决方案：对资产只断开引用链，由 `Resources.UnloadAsset` 或 ResourceManager 负责释放。

### 3. 活跃 SkeletonDataAsset vs 缓存池中的 SkeletonDataAsset

这是迭代中 v4 → v5 暴露的关键问题：只清理了 `_recycleDataAssets`（换装缓存池），漏掉了当前正在显示的 `_skeletonDataAsset`。两者**都需要**处理，且活跃的那个必须在 `skeletonGraphic.skeletonDataAsset = null` **之前**处理。

### 4. 纹理释放了但 Material 残留

v5 调用了 `Resources.UnloadAsset(tex)`，纹理已释放，但 Material 对象本身仍在内存中。需要追加 `Resources.UnloadAsset(mat)`（v6 修复）。

### 5. `_defaultMaterial` 持有额外引用

`_SetGray()` 方法会加载一个 UI spine 材质到 `_defaultMaterial` 字段。即使 atlas Material 被卸载，`_defaultMaterial` 仍持续引用（通常指向同一个 Material），导致 Material 无法释放。Dispose 中必须 `_defaultMaterial = null`。

### 6. CanvasRenderer 多槽 vs 单槽

仅设置 `skeletonGraphic.material = null` 只清除 `Graphic` 基类的默认材质槽（索引 0）。SkeletonGraphic 为多页 atlas 创建的额外槽位（索引 1+）不会被处理。必须遍历 `canvasRenderer.materialCount` 逐槽清理。

## ResourceManager 层面注意事项

### `AssetBundle.UnloadAsync(false)` 的语义

当前项目 ResourceManager 使用：

```csharp
assetBundle.UnloadAsync(false);
// false = 卸载 bundle 的序列化数据，但保留所有已加载的对象
```

这意味着即使引用计数降到 0、整个卸载流程走完（约 15-20 秒），atlas 纹理等已加载资源**仍然保留**在内存中。这是设计行为，不是 bug——`UnloadAsync(true)` 会销毁所有从 bundle 加载的对象，可能导致其他系统 Missing 引用。

**建议**：
- 如果 Spine 资源独立打包（一个角色一个 bundle），可考虑对此类 bundle 使用 `UnloadAsync(true)`
- 否则依赖上述 Dispose 中的 `Resources.UnloadAsset` 显式释放

### Editor 模式下的 `Resources.UnloadAsset`

当前项目在 Editor 非 AssetBundle 模式下，`AssetManager.Unload()` 中的 `Resources.UnloadAsset()` 调用被注释掉了。这意味着 Editor Play 时，Spine 资源**永不释放**，每次打开/关闭面板就累积一次。

修复后，`SpineUIController.Dispose()` 内部直接调用 `Resources.UnloadAsset()`，绕过了 ResourceManager 的 Editor 模式缺陷。

### 两段延迟

ResourceManager 卸载有双重延迟：
1. AssetProxy 在 refCount ≤ 0 后等待 ~10 秒才清理
2. BundleProxy 等待 ~5-10 秒才调用 `assetBundle.UnloadAsync`

这意味着 C# 层 Dispose 后，Bundle 层面的释放可能在 15-20 秒后才发生。`Dispose()` 中的 `Resources.UnloadAsset` 提供了**即时释放**通道。

## Lua 层注意事项

Lua 层的 `Clear()` 和 `Close()` 未主动调用 `SpineUIController.Dispose()`：

```lua
-- 当前实现
function UISpine:Clear()
    self.assetPath = nil       -- 仅清空 Lua 引用
    self:SetActive(false)      -- 仅隐藏 GameObject
end
```

**两种场景**：

| 场景 | 行为 | 是否需要改 |
|------|------|-----------|
| 面板销毁（`Destroy → OnDestroy → Dispose`） | 自动触发完整清理链 | 不需要 |
| 面板缓存（仅 Hide，不 Destroy） | GameObject 存活，Spine 资源一直占用 | **需要**：缓存时主动调 `Dispose()`，下次打开时重新加载 |

> 对于被缓存的 UI 面板（如英雄立绘），建议在 `Clear()` 中调用 C# 的 `SpineUIController.Dispose()`，并在下次激活时重新 `Initialize`。这样可以在面板不显示期间释放 GPU 纹理内存。

## 验证方法

确认资源已被释放：

1. **Unity Profiler → Memory → Assets**：搜索 Spine 相关 Texture2D/Material，确认数量在 Dispose 后减少
2. **`Resources.FindObjectsOfTypeAll<Texture2D>()`**：在 Dispose 前后对比纹理数量
3. **`Resources.FindObjectsOfTypeAll<Material>()`**：确认 Material 数量下降
4. **Memory Profiler 包**（推荐）：快照对比 Dispose 前后，定位残留引用的持有者

## 参见

- `[[concepts/Unity动画与Spine]]` — Spine 组件基础（SkeletonGraphic、SkeletonAnimation、Material 换装）
- `[[concepts/Unity资源管理]]` — AssetBundle 加载/卸载、Unity 内存模型、引用计数机制
- `[[concepts/CSharp内存GC]]` — IDisposable 模式、托管/原生内存区分
