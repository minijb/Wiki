---
title: "Unity 资源管理"
type: concept
updated: 2026-05-11
tags: [unity, asset-management, assetbundle, memory]
---

# Unity 资源管理

Unity 资源管理涉及三块独立的内存区域（托管、原生、非托管）及其不同的生命周期规则。资源从磁盘经 AssetBundle 管线加载到内存，再被实例化到场景中；每一步都需要相应的卸载策略，否则导致内存泄漏或 Missing 引用。理解 Asset、GameObject、AssetBundle 三者之间的关系，以及 `Instantiate` 的浅拷贝语义，是正确管理资源的基石。

## Unity 内存模型

Unity 运行时内存分为三个独立区域，各自由不同子系统管理：

| 内存类型 | 管理者 | 内容 | 释放方式 |
|---------|--------|------|---------|
| 托管内存 | Mono/IL2CPP GC | C# 对象（GameObject、MonoBehaviour、List 等） | GC 自动回收 |
| 原生内存 | Unity C++ 引擎 | Texture 像素、Mesh 顶点、Material 数据 | `Resources.UnloadAsset` / `UnloadUnusedAssets` |
| 非托管内存 | 开发者 | `NativeArray<T>`、`NativeList<T>` 等 | 显式 `Dispose()` |

> [!warning] 原生 ≠ 非托管
> 原生内存由 Unity C++ 引擎内部管理，开发者通过 C# API 间接控制。非托管内存通过 `NativeArray<T>` 等类型手动 `Dispose`。两者管理机制完全不同。

一个 `Texture2D` 同时占用托管内存（C# 包装器对象，含 `m_CachedPtr` 指向原生数据）和原生内存（像素数据）。`Resources.UnloadAsset` 只释放原生数据，C# 包装器留待 GC 回收。

### ScriptableObject 的拷贝语义

| 操作 | 值类型字段 | 原生资源引用 | 适用场景 |
|------|-----------|-------------|---------|
| 直接引用共享 | 共享同一对象 | 共享同一原生对象 | 多对象共用同一配置 |
| 浅拷贝 `Instantiate(so)` | 复制 | 共享（引用计数 +1） | 需要独立配置但共享资源 |
| 深拷贝 + `Instantiate(so.texture)` | 复制 | 完全独立副本 | 需要运行时修改且不能影响其他引用者 |

## AssetBundle 构建管线

### 构建输出

| 文件 | 内容 |
|------|------|
| `<bundlename>` | 资源序列化数据（主要加载目标） |
| `<bundlename>.manifest` | 依赖关系、资源列表、CRC 校验 |
| `<platform>` 主 Bundle | 包含所有 Bundle 依赖信息，提供 `AssetBundleManifest` |

### 压缩方式

| 方式 | 压缩率 | 加载特点 | 推荐场景 |
|------|--------|---------|---------|
| LZMA | 最高 | 需先解压整个 Bundle，峰值内存高 | 不推荐 |
| LZ4 | 中等 | 按块解压，按需加载单个资源，无额外解压缓冲 | **推荐（生产默认）** |
| 无压缩 | 无 | 加载最快，文件最大（2-5x） | 极端加载时间要求 |

```csharp
BuildPipeline.BuildAssetBundles(outputPath,
    BuildAssetBundleOptions.ChunkBasedCompression,  // LZ4
    BuildTarget.StandaloneWindows64);
```

### 打包策略

| 策略 | 粒度 | 优点 | 缺点 |
|------|------|------|------|
| 按目录 | 粗 | 简单直观，I/O 少 | 粒度不可控 |
| 按文件 | 细 | 热更新精准 | Bundle 数量多，依赖冗余 |
| 按类型 | 中 | 适合全局共享资源 | 更新粒度粗糙 |
| 混合 | — | 兼顾效率和灵活性 | 配置复杂 |

推荐：Shader/字体/公共 UI 按类型打包（低频更新）；功能模块独立 Bundle（热更友好）；大纹理/音频独立 Bundle（按需加载）。

## 运行时加载

### 加载方法对比

| 方法 | 适用场景 | 内存开销 | 卸载方式 |
|------|---------|---------|---------|
| `Resources.Load` | 原型/小项目 | 原生 + 托管 | `Resources.UnloadAsset` |
| `AssetBundle.LoadFromFile` | 生产环境 | Bundle 文件镜像（LZ4 无需额外缓冲） | `Unload(true/false)` |
| `AssetBundle.LoadFromMemory` | 加密/网络包 | 解压副本 + Bundle 镜像 | `Unload(true/false)` |
| `LoadAsset<T>` | 提取资源 | 原生 + 托管 | `UnloadAsset` + bundle unload |
| `LoadAssetAsync<T>` | 大资源异步 | 同上，不阻塞主线程 | 同上 |
| `Instantiate(prefab)` | 场景对象 | 托管克隆（共享原生引用） | `Destroy` |

### `Instantiate` 行为详解

| 组件 | `Instantiate` 行为 | 内存类型 |
|------|-------------------|---------|
| GameObject | 克隆（独立实例） | 托管 |
| Transform | 克隆（位置/旋转/缩放独立） | 托管 |
| MeshFilter.mesh | **引用**（共享） | 原生 |
| MeshRenderer.material | **引用**（共享） | 原生 |
| Shader | **引用**（全局共享，永不复制） | 原生 |
| Texture2D | **引用**（通过材质间接） | 原生 |
| MonoBehaviour 数据 | 克隆（序列化字段独立） | 托管 |
| MonoBehaviour 代码（IL） | **引用**（共享） | 托管 |

### 依赖加载流程

```
主 Manifest Bundle → AssetBundleManifest
    → GetAllDependencies("target")
    → 按序加载所有依赖 Bundle
    → 加载目标 Bundle
    → LoadAsset 提取资源
```

同一个 Bundle **不能重复加载**，需 BundleManager 缓存已加载实例。依赖 Bundle **不允许循环依赖**——构建时通过 `HashSet` 检测，有循环应提取到公共 Bundle。

## 卸载策略

### 卸载顺序（必须严格遵守）

```
场景对象(GameObject)  →  资源实例(Asset)  →  Bundle 内存镜像  →  原生资源
      ↓                       ↓                    ↓
   Destroy()            UnloadAsset()        Unload(true/false)
```

### 关键方法

| 方法 | 作用 | 注意 |
|------|------|------|
| `Destroy(gameObject)` | 移除场景实例，释放托管克隆 | 不影响被引用的原生资源 |
| `Resources.UnloadAsset(asset)` | 释放指定资源的原生数据 | 调用前确保所有引用者已销毁 |
| `bundle.Unload(false)` | 只释放 Bundle 文件镜像 | **推荐：加载完立即调用**，已提取资源仍可用 |
| `bundle.Unload(true)` | 释放镜像 + 所有资源 | 无视引用计数 → Missing 风险 |
| `Resources.UnloadUnusedAssets()` | 全局扫描释放计数为零的资源 | 开销大，仅 Loading/场景切换时调用 |

### 最佳实践流程

```csharp
// 1. 加载
var bundle = AssetBundle.LoadFromFile(path);
var prefab = bundle.LoadAsset<GameObject>("MyPrefab");

// 2. 立即释放文件镜像
bundle.Unload(false);

// 3. 使用资源
var instance = Instantiate(prefab);

// 4. 销毁实例
Destroy(instance);

// 5. 释放资源原生数据
Resources.UnloadAsset(prefab);

// 6. 场景切换时全局清理
Resources.UnloadUnusedAssets();
```

> [!danger] `Unload(true)` 的危险用法
> 先 `Destroy` 所有使用该 Bundle 资源的 GameObject → 再 `Unload(true)` → 最后可选 `UnloadUnusedAssets`。顺序反了会导致粉红色 Missing 材质。

### 引用计数机制

引用计数增加的来源：场景中的 Component 引用、C# 变量持有引用、资源间引用（如 Material 引用 Texture）。

引用计数降为零时资源**不会自动释放**——等待 `UnloadUnusedAssets` 调用或场景切换。这是设计行为，不是 bug。

## 参见

- [[sources/Unity资源管理-摘要|来源摘要]] — 完整要点速查
- [[concepts/CSharp内存GC|C# 内存与 GC]] — 托管内存与 GC 回收机制
