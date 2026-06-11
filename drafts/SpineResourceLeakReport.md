---
status: refined
merged-to: drafts/Spine资源卸载指南.md
updated: 2026-06-11
---

# Spine 资源泄露分析报告

## 问题描述
SpineUIController 所在的 GameObject 已被销毁（Destroy），但其关联的 SkeletonDataAsset 以及 atlas 纹理（Texture2D）仍然驻留在内存中，未被释放。

---

## 根本原因总结

共发现 **4 个核心原因** 和 **2 个次要原因**，按影响严重程度排列：

| # | 类别 | 简述 |
|---|------|------|
| 1 | **核心** | `SpineUIController.Dispose()` 未清理 `SkeletonGraphic` 内部动态创建的 Material，导致 atlas 纹理被持续引用 |
| 2 | **核心** | `DestroyImmediate(SkeletonDataAsset)` 无法级联释放其依赖的 AtlasAsset / Texture2D |
| 3 | **重要** | `BundleProxy.UnLoadAsync()` 使用 `assetBundle.UnloadAsync(false)` — 显式保留已加载资源 |
| 4 | **重要** | Editor 模式下 `Resources.UnloadAsset` 被注释掉，资源永不释放 |
| 5 | 次要 | `boundsCache` 静态字典无限增长 |
| 6 | 次要 | Lua 层未主动触发 Spine 资源清理 |

---

## 详细分析

### 原因 1（核心）：`SkeletonGraphic` 内部 Material 未被清理

**文件**: `Client/Assets/Script/Core/Graphic/SpineUIController.cs`

**问题代码** — `Dispose()` 方法 (第 134-153 行):

```csharp
public void Dispose()
{
    if (skeletonGraphic != null)
    {
        skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;  // 仅取消事件订阅
    }
    foreach (var item in _recycleDataAssets)
    {
        ResourceManager.Inst.UnloadAsync(item.Key);
        DestroyImmediate(item.Value);
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

**分析**:
- `Dispose()` 对 `skeletonGraphic` 的操作**仅限于取消 `OnMeshAndMaterialsUpdated` 事件订阅**。
- 在 `RebuildSkeletonDataAsset()` (第 711-737 行) 中，`skeletonGraphic.Initialize(true)` 会在 Spine 运行时内部**动态创建 `Material` 实例**（包含对 atlas `Texture2D` 的引用），并赋值给 `CanvasRenderer`。
- 当 GameObject 被销毁时，Unity 对 `CanvasRenderer` 上动态创建的 Material 的自动清理**不可靠** — 这些 Material 以及它们引用的纹理可能仍然驻留在内存中。

**缺失的清理步骤**:
```
skeletonGraphic.skeletonDataAsset = null;   // 断开 SkeletonDataAsset 引用
skeletonGraphic.Clear();                     // 释放内部 Mesh/Material
// 销毁 skeletonGraphic 上动态创建的 Material 实例
```

**后果**: `SkeletonGraphic` 内部的 Material 持续持有 atlas Texture2D 的强引用 → 纹理无法被 GC 或 Unity 卸载。

---

### 原因 2（核心）：`DestroyImmediate(SkeletonDataAsset)` 不级联释放依赖资源

**问题代码** — `Dispose()` 第 143 行:

```csharp
DestroyImmediate(item.Value);  // item.Value 是 SkeletonDataAsset (ScriptableObject)
```

**分析**:
- `SkeletonDataAsset` 是一个 `ScriptableObject`，内部持有对 `AtlasAsset`（也是 ScriptableObject）的引用链：`SkeletonDataAsset → AtlasAsset → Material → Texture2D`。
- `UnityEngine.Object.DestroyImmediate()` 在 ScriptableObject 上**只销毁该对象本身的 C# 包装**，不会递归销毁其引用的其他 ScriptableObject（AtlasAsset）或它们引用的 Texture2D。
- AtlasAsset 和 Texture2D 是作为**依赖资源**由 ResourceManager 一起加载的。当 SkeletonDataAsset 被 Destroy 后，这些依赖资源仍然被 ResourceManager 的内部引用计数系统持有。

**后果**: Atlas 纹理作为独立资源存在，`DestroyImmediate` 不会释放它们 → 纹理泄露。

---

### 原因 3（重要）：`assetBundle.UnloadAsync(false)` 显式保留已加载资源

**文件**: `Client/Packages/com.hero.resourcemanager/AssetBundle/BundleProxy.cs`

**问题代码**:

```csharp
assetBundle.UnloadAsync(false);  // false = 保留所有已加载的对象在内存中
```

**分析**:
- 当 `SpineUIController.Dispose()` 调用 `ResourceManager.Inst.UnloadAsync(assetPath)` 后，资源不会立即卸载。
- 实际卸载流程有**两段延迟**：
  1. AssetProxy 在 refCount ≤ 0 后等待 **10 秒**才清理
  2. BundleProxy 在依赖的 AssetProxy 全部清理后再等待 **随机 5-10 秒**才调用 `assetBundle.UnloadAsync`
- 最关键的是：Unity 的 `AssetBundle.UnloadAsync(false)` 的 `false` 参数语义是：**"卸载 bundle 的序列化数据，但保留所有已加载到内存中的对象"**。这意味着即使整个引用计数清理流程走完（约 15-20 秒后），纹理等已加载资源仍然被**显式保留**在内存中。
- 要真正释放纹理内存，需要传入 `true`，但 `true` 会销毁从该 bundle 加载的**所有**对象（可能影响仍在使用的其他系统）。

**后果**: ResourceManager 的卸载机制本身就**设计为不释放已加载资源**（`false` 参数），atlas 纹理在 bundle 卸载后依然存在。

---

### 原因 4（重要）：Editor 模式下资源永不卸载

**文件**: `Client/Packages/com.hero.resourcemanager/Resource/AssetManager.cs`

**问题代码**:

```csharp
// Resources.UnloadAsset 被注释掉了
```

**分析**:
- 在 Editor 非 AssetBundle 模式下，`AssetManager.Unload()` 只做 refCount 递减，实际的 `Resources.UnloadAsset()` 调用**被注释掉了**。
- 这意味着在 Editor 中 Play 时，所有通过 ResourceManager 加载的 Spine SkeletonDataAsset 及其依赖纹理**永远不会被卸载**。
- 每次打开/关闭包含 Spine 立绘的 UI 面板，资源就会累积一次。

**后果**: 开发阶段 Editor 运行时的内存持续增长，最终可能导致 Editor 崩溃。

---

### 原因 5（次要）：静态缓存 `boundsCache` 不清理

**文件**: `Client/Assets/Script/Core/Graphic/SpineUIController.cs` 第 29 行

```csharp
private static Dictionary<string, Vector2> boundsCache = new Dictionary<string, Vector2>();
```

- 每个唯一 Spine 资源的 bounds 信息被缓存在此静态字典中，永不清空。
- 虽然每个条目只有 16 字节（两个 float），但对于有数百个角色的项目，这是一个缓慢增长的内存占用。

---

### 原因 6（次要）：Lua 层未主动释放 Spine 资源

**文件**: 
- `Client/Assets/Script/Lua/Graphic/UISpine.lua` (第 46-49 行)
- `Client/Assets/Script/Lua/Client/UI/HeroSysPanel/HeroAvatarSimple/HeroAvatarSimple.lua` (第 59-65 行)

**UISpine:Clear()**:
```lua
function UISpine:Clear()
    self.assetPath = nil       -- 仅清空 Lua 引用
    self:SetActive(false)      -- 仅隐藏 GameObject
end
```

**HeroAvatarSimple:Close()**:
```lua
function HeroAvatarSimple:Close(delay)
    if self.visCharBust ~= nil then
        CharBustUtil.ReleaseVisCharBust(self.visCharBust)  -- 只释放 CharBust
        self.visCharBust = nil
    end
    HeroAvatarSimple.super.Close(self, delay)  -- 仅 Hide + unloadImage
end
```

- Lua 层的 `Clear()` 和 `Close()` 均未调用 `SpineUIController.Dispose()` 或触发 Spine 资源的显式清理。
- 虽然最终 `Control:destroy()` → `GameObject.DestroyImmediate` → `OnDestroy` → `Dispose()` 链会触发 C# 端清理，但在此之前（尤其是面板被缓存而非销毁时），资源一直占用内存。

---

## 资源生命周期完整链路

```
Lua 层:
  HeroAvatarSimple:destroy()
    → Control:destroy()
        → GameObject.DestroyImmediate(self.gameObject)   ← 销毁 GameObject
        → resMgr:UnloadAsync(self.assetPath, handle)     ← 卸载 Control 自己的 prefab
           (不包括 Spine SkeletonDataAsset 路径)

Unity 层:
  DestroyImmediate(GameObject)
    → SpineUIController.OnDestroy()
        → Dispose()
            → skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild  ← 取消事件
            → ResourceManager.Inst.UnloadAsync(assetPath)                  ← 异步+延迟
            → DestroyImmediate(SkeletonDataAsset)                          ← 不级联
            → 未设置 skeletonGraphic.skeletonDataAsset = null
            → 未调用 skeletonGraphic.Clear()
            → 未销毁 SkeletonGraphic 动态创建的 Material

ResourceManager 层:
  UnloadAsync(assetPath)
    → AssetProxy.ReduceRef()
    → 等待 10 秒...
    → AssetProxy.UnLoad()
        → BundleProxy.ReduceRef()   (级联依赖 bundle)
        → 等待 5-10 秒...
        → assetBundle.UnloadAsync(false)   ← false = 保留已加载资源!
    → Editor 模式: Resources.UnloadAsset 被注释  ← 永不释放!
```

---

## 问题总结

GameObject 被销毁后资源仍然存在的**直接原因**：

1. `SkeletonGraphic` 内动态创建的 Material 未被销毁 → 纹理被持续引用
2. `DestroyImmediate(SkeletonDataAsset)` 不释放依赖的 AtlasAsset/Texture2D
3. ResourceManager 的 AssetBundle 卸载使用 `UnloadAsync(false)` → 纹理被显式保留
4. Editor 模式下 `Resources.UnloadAsset` 被注释 → 资源永不释放

**这四条原因形成了"完美泄露链"**：即使最上层（Lua/C# Dispose）正确触发，底层（ResourceManager/AssetBundle）的设计也使得纹理资源无法被真正释放。

---

## 修复记录

### 最终版 `Dispose()` 方法

**文件**: `Client/Assets/Script/Core/Graphic/SpineUIController.cs`

```csharp
public void Dispose()
{
    if (skeletonGraphic != null)
    {
        skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;

        // 步骤1：先释放当前 SkeletonDataAsset 引用的 atlas 纹理
        // 顺序关键 — 必须在 skeletonGraphic.skeletonDataAsset = null 之前访问 _skeletonDataAsset
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

    foreach (var item in _recycleDataAssets)
    {
        // 断开 SkeletonDataAsset → AtlasAsset 引用链
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

### 三步清理详解

**问题链路**:
```
SkeletonGraphic.Initialize(true)
  → 创建 Material 实例 / 引用 atlas Material
  → Material 持有 mainTexture → Texture2D
  → CanvasRenderer 持有 Material[]
  → GameObject 销毁后 Material 和 Texture2D 残留
```

| 步骤 | 代码 | 原因 |
|------|------|------|
| **1. 卸载 atlas 纹理** | 遍历 `_skeletonDataAsset.atlasAssets[].Materials[].mainTexture` → `mat.mainTexture = null` → `Resources.UnloadAsset(tex)` | 必须在 `skeletonDataAsset = null` **之前**执行。Texture2D 是独立于 Material 的 Unity 资产，即使 Material 被销毁，纹理仍需显式卸载。`Resources.UnloadAsset` 立即释放纹理的 CPU/GPU 内存 |
| **2. 清理 SkeletonGraphic** | `skeletonDataAsset = null` + `Clear()` + `material = null` | 释放 Spine 运行时内部 Mesh、骨骼状态，断开 C# 层面的 data asset 引用 |
| **3. 清理 CanvasRenderer** | 遍历 `materialCount`，逐槽 `GetMaterial(i)` → `SetMaterial(null, i)` → `mat.mainTexture = null` | `SkeletonGraphic` 可能为多页 atlas 创建多个材质槽。仅设 `material = null` 只清除 Graphic 基类的默认材质。不调用 `DestroyImmediate(mat)` — Material 可能是 AssetBundle 加载的资产引用 |

### SkeletonDataAsset 引用链断开

| 代码 | 原因 |
|------|------|
| `item.Value.atlasAssets = null` | `SkeletonDataAsset.atlasAssets` 持有 `AtlasAsset[]`，AtlasAsset 持有 Material → Texture2D。将字段置 null 断开引用链 |
| 移除 `DestroyImmediate(item.Value)` | 对 Asset 类型调用 `DestroyImmediate` 在运行时抛 `"Destroying assets is not permitted"`；ResourceManager 的 `UnloadAsync` 已通过引用计数管理生命周期 |
| `_skeletonDataAsset = null; _skeletonData = null` | 清空控制器自身字段，避免悬空引用 |


## 迭代修复过程

详见独立报告：[SpineFixIterations.md](SpineFixIterations.md)，记录了从初版到最终版的 5 次迭代：每次的完整代码改动、验证结果和失败原因。

