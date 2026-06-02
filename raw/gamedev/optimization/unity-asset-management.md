---
title: Unity 资源管理与 AssetBundle
type: source
updated: 2026-06-02
tags:
  - unity
  - asset-management
  - assetbundle
  - memory
---

# Unity 资源管理与 AssetBundle

Unity 的资源管理是一个横跨内存模型、加载管线、构建流程和运行时架构的复杂系统。理解它的关键不在于记住 API，而在于理解一个核心事实：**Unity 的内存不是一块，而是三块**——托管内存、原生内存和非托管内存各自由不同的子系统管理，遵循不同的生命周期规则。AssetBundle 则是这三块内存之间的桥梁：它将磁盘上的序列化资源加载到原生内存，再根据需要将引用传递到托管层。

本文从内存模型出发，逐层深入加载流程、AssetBundle 构建管线、运行时架构，最终提炼面试和工程决策中的关键要点。

## Unity 内存模型

### 三种内存类型

Unity 运行时的内存分为三个独立的区域。每一块由不同的运行时管理，使用不同的分配和回收策略。不理解这三块的区别，就无从理解资源加载和卸载的完整行为。

**托管内存（Managed Memory）** 由 Mono/IL2CPP 虚拟机中的垃圾回收器（GC）管理。所有 C# 对象——`GameObject`、`MonoBehaviour` 实例、`List<T>`、以及值类型的装箱副本——都分配在这里。托管内存的生命周期由 GC 决定：对象不再被任何根引用可达时标记为垃圾，GC 在合适的时机回收。开发者可以通过 `GC.Collect()` 强制触发回收，但这通常不是推荐做法。

**原生内存（Native Memory）** 由 Unity 引擎的 C++ 层管理。`Texture2D`、`Mesh`、`Material`、`Shader`、`AnimationClip`、`AudioClip` 等资源的主体数据都存储在原生内存中。C# 层的 `Texture2D` 对象只是一个小小的包装器——它包含一个 `m_CachedPtr` 整数字段，指向 C++ 层真正的纹理数据。当你调用 `Resources.UnloadAsset(texture)` 时，释放的是 C++ 层的数据，C# 层的包装器对象仍然存在（变成一个空壳），直到 GC 回收它。

**非托管内存（Unmanaged Memory）** 由开发者手动管理。`NativeArray<T>`、`NativeList<T>`、`NativeHashMap<TKey, TValue>` 等 Unity Collections 包中的容器分配在这里。这些容器必须显式调用 `Dispose()` 释放，否则造成内存泄漏。非托管内存的优势在于它不受 GC 影响——不会触发 GC 暂停，也不会因为 GC 压缩而移动——适合 Job System 和 Burst 编译的高性能路径。

> [!warning] 原生内存不是非托管内存
> 原生内存是 Unity C++ 引擎内部管理的（如 Texture 像素数据、Mesh 顶点缓冲），开发者通过 C# API 间接控制。非托管内存是通过 `NativeArray<T>` 等类型手动 `Dispose` 的内存。两者名称相似但管理机制完全不同。

这三块内存的交互是资源管理的核心。一个 `Texture2D` 对象跨越了托管内存和原生内存：C# 对象在托管堆上，像素数据在原生内存中。当你销毁这个纹理时，必须同时处理这两块——`Resources.UnloadAsset` 释放原生数据，而托管对象等 GC 回收。如果只销毁 C# 对象（设为 null），原生数据仍然占用内存，直到调用 `UnloadUnusedAssets` 或场景切换。

### ScriptableObject 与 Native Memory

`ScriptableObject`（SO）是 Unity 中最常用的数据容器，但它的内存模型经常被误解。一个 SO 实例内部包含两种类型的数据：

- **值类型字段**（`int`、`float`、`bool`、`Vector3` 等）直接存储在托管堆上的 C# 对象内部，随 SO 对象一起被 GC 管理。
- **原生资源引用**（`Texture2D`、`Mesh`、`Material`、`Sprite` 等）在 C# 层只存储一个 `m_CachedPtr`（指向 C++ 层原生对象的指针）和一个引用计数。实际数据在原生内存中。

这意味着当你将同一个 `Texture2D` 赋值给多个 `ScriptableObject` 的字段时，它们共享的是同一个原生纹理数据。C# 层有多个引用指向同一个原生对象。修改其中一个 SO 中的纹理引用不会影响其他 SO，但如果通过某个引用修改了纹理内容（如 `SetPixels`），所有引用这个纹理的对象都会看到变化——因为它们指向的是同一块原生内存。

SO 的 `Instantiate` 行为也建立在这个模型之上：

1. **直接引用共享（不拷贝）**：将同一个 SO 赋值给多个引用变量，它们指向内存中的同一个 C# 对象和同一个原生资源。

2. **浅拷贝（Shallow Clone）**：调用 `Instantiate(originalSO)` 会创建一个新的 SO 实例（不同的 `GetInstanceID()`），值类型字段被复制。但原生资源引用字段——`Texture2D`、`Mesh` 等——仍然指向原始的原生对象。C++ 层的引用计数增加，但数据不复制。

3. **深拷贝（Deep Clone）**：对 SO 执行 `Instantiate` 后，再对每个原生资源引用字段调用 `Instantiate`（如 `Instantiate(so.texture)`），创建完全独立的原生数据副本。这会消耗额外的原生内存，但两个 SO 从此完全独立，修改互不影响。

```csharp
// 浅拷贝：两个 SO 共享同一个 Texture
var shallowCopy = Instantiate(originalSO);
// shallowCopy.texture == originalSO.texture → 同一个原生纹理

// 深拷贝：完全独立
var deepCopy = Instantiate(originalSO);
deepCopy.texture = Instantiate(originalSO.texture);
// 现在两个 SO 使用不同的纹理数据
```

> [!tip] 内存成本估算
> 一个 `ScriptableObject` 自身在托管堆上通常只占几十到几百字节。真正的内存开销来自它引用的原生资源——一个 2048×2048 的 RGBA32 纹理占用 16MB 原生内存。设计 SO 结构时，关注的是原生资源的共享和复制，而不是 SO 对象本身的内存。

### 资源拷贝语义

Unity 中资源的"拷贝"有三种不同的语义，每种对应的内存开销和副作用不同：

**引用共享（Reference Sharing）** 是最常见的形式。多个 `GameObject` 引用同一个 `Material`、同一个 `Mesh`、同一个 `Texture`。原生内存中只有一份数据。这是运行时默认行为，也是内存效率最高的方式。

**实例化复制（Instantiate Copy）** 发生在 `Instantiate(prefab)` 时。`Transform` 和 `GameObject` 自身被真正复制——每个实例有独立的 `Transform` 组件和独立的位置、旋转、缩放。但 `MeshFilter` 中的 `Mesh` 和 `MeshRenderer` 中的 `Material` 仍然是引用——所有实例共享同一个网格和材质。只有 `Script` 组件上的序列化数据被复制（每个实例有独立的字段值），代码（IL）仍然是共享引用。

**显示复制（Explicit Copy）** 通过 `Instantiate(texture)` 或 `Instantiate(mesh)` 创建原生资源的真正副本。这会分配新的原生内存来存储像素数据或顶点数据。通常只在运行时需要修改资源且不能影响其他引用者时使用。

## 资源加载与卸载

### 资源加载流程

Unity 资源加载的核心路径是：磁盘 → AssetBundle 内存镜像 → 原生内存（资源数据） → 托管内存（C# 包装器） → 场景对象。

1. **加载 AssetBundle 文件**：`AssetBundle.LoadFromFile(path)` 将磁盘上的 AssetBundle 文件读入一块称为"bundle 内存镜像"的区域。这不是资源的实际数据，而是序列化的二进制块。使用 LZ4 压缩时，这个镜像支持按需解压单个资源块；使用 LZMA 压缩时，整个镜像必须先全部解压才能访问任何资源。

2. **从 Bundle 加载资源**：`AssetBundle.LoadAsset<T>(name)` 或 `Resources.Load<T>(path)` 将指定的资源从 bundle 镜像中提取出来，在原生内存中创建实际的纹理/网格/材质数据，并在托管堆上创建对应的 C# 包装器对象。

3. **实例化到场景**：`Instantiate(prefab)` 以加载的资源为模板，创建 `GameObject` 及其 Components 的副本。这一步分配的是托管内存（GameObjects、Components）和对原生资源的引用。

每一步都增加内存占用，且每一步的释放方式不同——这是资源泄漏最常见的根源。

### Prefab 内部结构

理解 `Instantiate(prefab)` 时到底发生了什么，是避免内存问题的关键。以包含 `MeshRenderer` 和自定义 `MonoBehaviour` 脚本的 Prefab 为例：

| 组件 | Instantiate 行为 | 内存类型 |
|------|-----------------|---------|
| `GameObject` | 克隆（新建独立实例） | 托管内存 |
| `Transform` | 克隆（位置/旋转/缩放独立） | 托管内存 |
| `MeshFilter.mesh` | **引用**（所有实例共享同一网格） | 原生内存 |
| `MeshRenderer.material` | **引用**（默认共享材质） | 原生内存 |
| `Shader` | **引用**（全局共享，永远不复制） | 原生内存 |
| `Texture2D`（材质的纹理） | **引用**（通过材质间接引用） | 原生内存 |
| `MonoBehaviour` 脚本数据 | **克隆**（序列化字段值独立复制） | 托管内存 |
| `MonoBehaviour` 脚本代码 | **引用**（IL 代码全局共享） | 托管内存 |

> [!important] Destroy 只移除克隆
> 调用 `Destroy(gameObject)` 时，释放的是 `GameObject` 和 `Transform` 的克隆（托管内存），以及 `MonoBehaviour` 的序列化数据副本。它对 `Mesh`、`Material`、`Texture`、`Shader` 的原生数据**没有任何影响**——这些是引用，不是副本。只有当没有任何对象引用这些资源时，`Resources.UnloadUnusedAssets()` 才会释放它们。

### 加载方法对比

Unity 提供多种资源加载方式，各有不同的适用场景和内存行为：

**`Resources.Load<T>(path)`**：从 `Resources` 文件夹加载资源。简单直接，但 `Resources` 文件夹中的所有资源在应用启动时被编入索引（红黑树结构），且 `Resources` 中的资源不能被动态卸载——它们要么一直占用内存，要么永远无法被 `AssetBundle.Unload(true)` 清理。适合原型阶段和少量全局资源，不适合大型项目。

**`AssetBundle.LoadFromFile(path)`**：从磁盘文件加载 AssetBundle。这是推荐的生产环境方式。加载的是 bundle 的文件镜像（压缩的二进制块），不额外分配解压缓冲（LZ4 支持按需解压）。

**`AssetBundle.LoadFromMemory(byte[])`**：从字节数组加载 AssetBundle。加载的是整个 bundle 的解压副本，占用额外的原生内存。仅在没有文件系统访问权限（如加密包）时使用。

**`AssetBundle.LoadAsset<T>(name)`**：从已加载的 AssetBundle 中提取指定资源。创建原生数据副本（从 bundle 镜像中解压）和 C# 包装器。

**`AssetBundle.LoadAssetAsync<T>(name)`**：异步版本。返回 `AssetBundleRequest`，通过协程或 `Completed` 回调获取结果。适合需要保持帧率的大资源加载。

**`Instantiate(prefab)`**：以已加载资源为模板创建场景对象。这是浅拷贝——克隆 GameObject/Transform/脚本数据，引用网格/材质/纹理。

| 方法 | 适用场景 | 内存开销 | 卸载方式 |
|------|---------|---------|---------|
| `Resources.Load` | 原型/小项目 | 原生+托管 | `Resources.UnloadAsset` |
| `LoadFromFile` | 生产环境 | bundle 镜像 | `Unload(true/false)` |
| `LoadFromMemory` | 加密/网络包 | 解压副本+bundle 镜像 | `Unload(true/false)` |
| `LoadAsset` | 提取资源 | 原生+托管 | `UnloadAsset` + bundle unload |
| `Instantiate` | 场景对象 | 托管克隆 | `Destroy` |

### 卸载策略

资源卸载是 Unity 内存管理中最容易出错的环节。错误的卸载顺序会导致三种后果：**内存泄漏**（该释放的内存没释放）、**引用丢失**（不该释放的资源被释放）、**Missing 引用**（场景中出现粉红色材质/Missing Script）。

正确的卸载策略必须遵循一个严格的**依赖方向**：从最外层的派生对象向内层的共享资源逐层释放。

```
场景对象(GameObject) → 资源实例(Asset) → Bundle 内存镜像 → 原生资源
       ↓                    ↓                    ↓
    Destroy()          UnloadAsset()        Unload(true/false)
```

**`Destroy(gameObject)`**：移除场景中的 GameObject 及其克隆的组件数据。不影响被引用的资源（Material、Mesh、Texture）。

**`Resources.UnloadAsset(asset)`**：释放指定资源的原生数据。如果该资源仍被场景中的对象引用，这些引用会变成 Missing。调用前必须确保所有引用者已被销毁。

**`AssetBundle.Unload(false)`**：释放 AssetBundle 的文件内存镜像。已通过 `LoadAsset` 提取的资源不受影响，仍然可用。这是**推荐做法**——加载完成后立即 `Unload(false)`，释放文件镜像，保留资源。

**`AssetBundle.Unload(true)`**：释放 AssetBundle 的文件内存镜像**同时**强制释放所有从该 Bundle 加载的资源。任何仍持有这些资源引用的对象会变成 Missing（粉红色材质）。调用前必须确保所有使用该 Bundle 资源的场景对象已被销毁。

**`Resources.UnloadUnusedAssets()`**：扫描所有资源，释放引用计数为零的原生资源。这是一个**全局操作**，开销较大，通常在场景切换或 Loading 界面时调用。

> [!danger] Unload(true) 的危险用法
> `Unload(true)` 会强制释放资源，无视引用计数。如果场景中仍有对象引用这些资源，它们不会消失——而是变成 Missing 状态。正确的顺序是：先 `Destroy` 所有使用该 Bundle 资源的 GameObject → 再 `Unload(true)` → 最后可选 `UnloadUnusedAssets`。

**最佳实践流程**：

```csharp
// 1. 加载 Bundle
var bundle = AssetBundle.LoadFromFile(path);

// 2. 提取需要的资源
var prefab = bundle.LoadAsset<GameObject>("MyPrefab");
var texture = bundle.LoadAsset<Texture2D>("MyTexture");

// 3. 立即释放文件镜像（推荐）
bundle.Unload(false);

// 4. 使用资源
var instance = Instantiate(prefab);

// 5. 不再需要时销毁实例
Destroy(instance);

// 6. 释放资源
Resources.UnloadAsset(prefab);
Resources.UnloadAsset(texture);

// 7. 场景切换时全局清理
Resources.UnloadUnusedAssets();
```

### UnloadUnusedAssets 原理

`Resources.UnloadUnusedAssets()` 的工作机制是**引用计数检查**。Unity 内部为每个原生资源维护一个引用计数。以下情况会增加引用计数：

- 资源被场景中的某个 Component 引用（如 `MeshRenderer.material`）
- 资源被 C# 变量引用（如 `public Texture2D myTexture;`）
- 资源被另一个资源引用（如 Material 引用 Texture）

当引用计数**同时**满足以下条件时，资源被判定为"未使用"：

1. **没有场景引用**：没有任何场景中的 GameObject 或 Component 直接或间接引用该资源。
2. **没有变量引用**：没有任何存活 C# 对象的字段持有该资源的引用。
3. **AssetBundle 已释放或引用关系已解除**：对于从 Bundle 加载的资源，要么 Bundle 已通过 `Unload(true)` 释放，要么 `Unload(false)` 后资源的引用计数独立管理。

`UnloadUnusedAssets` 的开销很大——它必须扫描所有已加载的资源，检查每个资源的引用计数。在大型项目中，这个操作可能需要数百毫秒。因此应该在 Loading 界面或场景切换的过渡阶段调用，而非在游戏过程中频繁调用。

> [!tip] 引用计数为零不一定立即释放
> 资源的引用计数降到零后，资源不会自动释放——它等待 `UnloadUnusedAssets` 的调用或场景切换。这就是为什么切换场景时 Unity 会自动执行等效的清理操作。

## AssetBundle 基础

### 构建与压缩

AssetBundle 的构建通过 `BuildPipeline.BuildAssetBundles` 完成。开发者通常使用 **AssetBundle Browser** 工具或自定义脚本来配置哪些资源打入哪些 Bundle。

**构建输出**包含三类文件：

1. **资源文件**（`<bundlename>`）：实际包含资源数据的序列化文件。这是加载的主要目标。
2. **Manifest 文件**（`<bundlename>.manifest`）：文本文件，记录了 Bundle 的依赖关系、资源列表、CRC 校验值。
3. **主 Bundle**（`<platformname>` 或 `<outputdir>` 同名文件）：包含所有 Bundle 的依赖信息的 AssetBundle，其 Manifest 文件提供了 `AssetBundleManifest` 对象用于运行时解析依赖。

**压缩方式**选择直接影响加载性能和内存占用：

**LZMA（默认压缩）**：对整个 Bundle 进行流式压缩。压缩率最高，文件最小。但加载时需要**先解压整个 Bundle**，然后才能访问任何资源。内存中同时存在压缩数据和解压数据，峰值内存较高。不适合大型 Bundle。

**LZ4（推荐）**：分块压缩。每个资源块独立压缩，可以按需解压单个资源而不需要解压整个 Bundle。加载速度显著快于 LZMA，且不需要额外的解压缓冲区。内存占用更可控。Unity 2017.3+ 的默认推荐方式。

**不压缩（Uncompressed）**：加载最快，无解压开销，但文件最大（通常大 2-5 倍）。适合极少数对加载时间要求极端苛刻的场景。

```csharp
// 构建时指定压缩方式
BuildPipeline.BuildAssetBundles(outputPath,
    BuildAssetBundleOptions.ChunkBasedCompression,  // LZ4
    BuildTarget.StandaloneWindows64);
```

> [!tip] 构建选项
> `ClearFolder` 在构建前清空输出目录。`CopyToStreamingAssets` 将构建产物复制到 `StreamingAssets` 文件夹，使其随应用一起发布。`ForceRebuildAssetBundle` 强制重建所有 Bundle（忽略增量构建缓存）。

### 同步/异步加载

**同步加载**使用 `AssetBundle.LoadFromFile` + `LoadAsset<T>`：

```csharp
var bundle = AssetBundle.LoadFromFile(bundlePath);
var prefab = bundle.LoadAsset<GameObject>("MyPrefab");
// 加载期间主线程阻塞，帧率下降
```

适合小型 Bundle、Loading 界面期间、或不关心帧率的场景。

**异步加载**使用 `LoadAssetAsync<T>` 配合协程：

```csharp
IEnumerator LoadAsync()
{
    var bundleLoad = AssetBundle.LoadFromFileAsync(bundlePath);
    yield return bundleLoad;
    var bundle = bundleLoad.assetBundle;

    var request = bundle.LoadAssetAsync<GameObject>("MyPrefab");
    yield return request;
    var prefab = request.asset as GameObject;
}
```

异步加载的核心优势是**不阻塞主线程**——在资源解压和原生内存分配期间，Unity 可以继续渲染帧，保持流畅的 Loading 画面或游戏体验。`AssetBundleRequest` 可以设置 `priority` 来控制加载顺序，设置 `allowSceneActivation` 来控制场景激活时机。

对于类型不固定的资源，可以使用非泛型版本：

```csharp
var request = bundle.LoadAssetAsync("MyAsset");
// request.asset 的类型在运行时确定
```

同一个 AssetBundle **不能被重复加载**。如果尝试加载一个已经处于加载状态的 Bundle，Unity 会抛出错误。因此运行时需要一个管理器来追踪已加载的 Bundle。

### 依赖管理

AssetBundle 之间的依赖关系是自动处理的。如果 Bundle A 中的 Prefab 引用了 Bundle B 中的 Texture，构建管线会自动记录这个依赖关系。但运行时的加载顺序必须正确：**必须先加载被依赖的 Bundle**。

正确的加载流程：

1. 加载主 Bundle（平台 Manifest Bundle）
2. 从主 Bundle 提取 `AssetBundleManifest`
3. 查询目标 Bundle 的所有依赖
4. 按依赖顺序加载所有依赖 Bundle
5. 最后加载目标 Bundle

```csharp
// 加载主 Manifest Bundle
var mainBundle = AssetBundle.LoadFromFile(manifestPath);
var manifest = mainBundle.LoadAsset<AssetBundleManifest>("AssetBundleManifest");

// 查询依赖
string[] deps = manifest.GetAllDependencies("targetbundle");
foreach (var dep in deps)
{
    AssetBundle.LoadFromFile(Path.Combine(bundleDir, dep));
}

// 加载目标 Bundle
var targetBundle = AssetBundle.LoadFromFile(Path.Combine(bundleDir, "targetbundle"));
```

如果依赖的 Bundle 未加载，对目标 Bundle 调用 `LoadAsset` 时会失败——资源引用无法解析。

> [!warning] 循环依赖
> AssetBundle 之间**不允许循环依赖**。如果 Bundle A 依赖 Bundle B、Bundle B 依赖 Bundle A，构建会失败。构建管线使用 `HashSet` 追踪已处理的 Bundle 来检测循环。解决方案是将共享资源提取到第三个公共 Bundle 中。

### 管理器模式

对于任何非 trivial 的项目，手动管理 AssetBundle 的加载、依赖、卸载是不现实的。标准的做法是实现一个 **BundleManager** 类，封装以下职责：

- **缓存已加载的 Bundle**：使用 `Dictionary<string, AssetBundle>` 维护已加载 Bundle 的引用，避免重复加载。
- **依赖解析**：加载主 Manifest 后，对每个目标 Bundle 自动加载其所有依赖。
- **引用计数**：为每个 Bundle 维护加载计数。多个资源可能来自同一个 Bundle，只有当所有使用者都释放后才真正卸载 Bundle。
- **异步加载队列**：管理正在进行的异步加载操作，支持并发控制和优先级。
- **延迟卸载列表**：将标记为卸载的 Bundle 加入列表，在 `LateUpdate` 等合适时机批量处理。

```csharp
public class BundleManager
{
    private Dictionary<string, AssetBundle> _loadedBundles = new();
    private Dictionary<string, int> _refCounts = new();
    private AssetBundleManifest _manifest;

    public AssetBundle LoadBundle(string name)
    {
        if (_loadedBundles.TryGetValue(name, out var cached))
        {
            _refCounts[name]++;
            return cached;
        }

        // 加载所有依赖
        var deps = _manifest.GetAllDependencies(name);
        foreach (var dep in deps)
        {
            LoadBundle(dep);  // 递归加载依赖
        }

        var bundle = AssetBundle.LoadFromFile(GetBundlePath(name));
        _loadedBundles[name] = bundle;
        _refCounts[name] = 1;
        return bundle;
    }

    public void UnloadBundle(string name)
    {
        if (!_refCounts.ContainsKey(name)) return;
        _refCounts[name]--;
        if (_refCounts[name] <= 0)
        {
            _loadedBundles[name].Unload(false);
            _loadedBundles.Remove(name);
            _refCounts.Remove(name);
        }
    }
}
```

## AssetBundle 构建管线

### 文件收集与依赖分析

构建管线的第一步是确定哪些资源需要打包以及如何分组。这通常通过配置文件驱动——XML 或 JSON 格式的构建设置，描述文件路径规则、过滤条件和 Bundle 命名策略。

**文件收集** 使用 `Directory.GetFiles` 遍历指定目录，应用前缀/后缀过滤器和 ignore 路径列表：

```csharp
var files = Directory.GetFiles(assetPath, "*.*", SearchOption.AllDirectories)
    .Where(f => !f.EndsWith(".cs"))
    .Where(f => !f.EndsWith(".dll"))
    .Where(f => !ignorePaths.Any(p => f.StartsWith(p)))
    .ToArray();
```

脚本文件（`.cs`）和程序集文件（`.dll`）必须被排除——它们不是资源，不应该进入 AssetBundle。

**依赖分析** 是构建管线的核心环节。使用 `AssetDatabase.GetDependencies` 递归获取每个资源的所有依赖：

```csharp
string[] GetRecursiveDependencies(string assetPath)
{
    var deps = AssetDatabase.GetDependencies(assetPath);
    var result = new HashSet<string>();
    foreach (var dep in deps)
    {
        if (dep.EndsWith(".cs") || dep.EndsWith(".dll")) continue;
        if (result.Add(dep))
        {
            // 递归获取二级依赖
            var subDeps = AssetDatabase.GetDependencies(dep);
            foreach (var sub in subDeps)
            {
                if (!sub.EndsWith(".cs") && !sub.EndsWith(".dll"))
                    result.Add(sub);
            }
        }
    }
    return result.ToArray();
}
```

资源在依赖分析后被分为两类：

- **直接资源**：开发者明确标记或配置为需要打包的资源。这些资源会在目标 Bundle 中保留完整数据。
- **依赖资源**：被直接资源引用但本身未标记的资源。这些资源会被自动打包到依赖它们的 Bundle 中，或者被提取为独立的依赖 Bundle。

> [!important] 依赖资源的隐式打包
> 如果一个依赖资源被多个 Bundle 中的直接资源引用，它可能被打包到每个 Bundle 中（造成冗余），也可能被提取为独立 Bundle（需要额外的依赖加载）。这取决于构建配置——Unity 默认会将共享依赖打包到每个引用它的 Bundle 中。使用 `AssetBundleManifest.GetAllDependencies` 可以识别这些共享依赖并手动提取为独立 Bundle。

### Bundle 分类与命名

Bundle 的命名策略直接影响加载效率和热更新粒度。常见的分类方式：

**按目录打包（Directory）**：一个目录下的所有资源打成一个 Bundle。简单直观，但可能导致 Bundle 粒度过大或过小。

**按文件打包（File）**：每个资源独立成 Bundle。粒度最细，热更新最精准。但 Bundle 数量多，文件 I/O 次数多，且可能产生大量冗余依赖。

**按类型打包（All）**：所有指定类型的资源打成一个 Bundle。适合全局共享资源（Shader、字体、公共 UI）。

**混合策略**：大多数项目采用混合策略——大型场景按目录打包，UI 贴图按功能模块打包，Shader 和公共材质按类型打包。

Bundle 命名需要支持**最长路径匹配优先级**。当一个资源匹配多个打包规则时，应选择路径最具体（最长）的规则：

```csharp
BundleRule GetBuildItem(string assetPath)
{
    // 按路径长度降序排列规则，取第一个匹配的
    return rules
        .OrderByDescending(r => r.pathPrefix.Length)
        .FirstOrDefault(r => assetPath.StartsWith(r.pathPrefix));
}
```

### Manifest 生成

构建过程中生成的 Manifest 文件（不是 `AssetBundleManifest`，而是自定义的二进制 Manifest）记录了关键元数据，用于运行时的资源定位和加载：

- **资源 ID 映射**：每个资源在 Bundle 中的内部 ID 与其原始路径的对应关系
- **Bundle 信息**：每个 Bundle 的名称、CRC 校验值、文件大小
- **依赖信息**：每个 Bundle 依赖哪些其他 Bundle
- **版本信息**：用于热更新的版本号和文件哈希

这些信息在构建时通过二进制序列化写入临时文件，运行时加载后解析为内存中的数据结构。Manifest 是 BundleManager 和 ResourceManager 正确加载资源的依据。

## 运行时资源管理架构

### BundleManager

BundleManager 是运行时资源管理的核心——它负责 AssetBundle 的生命周期。一个完整的 BundleManager 至少包含以下组件：

**已加载 Bundle 字典**：`Dictionary<string, ABundle>`，以 Bundle 名为键，存储已加载的 Bundle 引用。每个 `ABundle` 对象封装了 `AssetBundle` 实例、加载状态（Loading/Loaded/Unloading）、引用计数。

**异步加载列表**：`List<ABundle>`，追踪正在进行异步加载的 Bundle。在 `Update` 或 `LateUpdate` 中检查是否完成，完成后移入已加载字典。

**延迟卸载列表**：`List<ABundle>`，存储引用计数已降为零但尚未执行 `Unload` 的 Bundle。在 `LateUpdate` 中批量处理，避免在加载过程中卸载依赖 Bundle。

**平台 Manifest**：从主 Bundle 加载的 `AssetBundleManifest`，用于依赖解析。

核心加载流程（`LoadInternal`）：

1. **缓存检查**：查询已加载字典，若已存在则增加引用计数并直接返回
2. **创建 ABundle**：根据配置选择同步或异步加载方式
3. **依赖加载**：通过 Manifest 查询所有依赖，递归调用 `LoadInternal` 加载每个依赖
4. **引用计数**：依赖加载完成后，增加目标 Bundle 的引用计数
5. **加载资源**：调用 `LoadAsset` 提取需要的资源

```csharp
ABundle LoadInternal(string bundleName, bool async)
{
    if (_loadedBundles.TryGetValue(bundleName, out var cached))
    {
        cached.AddReference();
        return cached;
    }

    // 递归加载依赖
    var deps = _manifest.GetAllDependencies(bundleName);
    foreach (var dep in deps)
    {
        LoadInternal(dep, async);
    }

    // 加载目标 Bundle
    var ab = async
        ? new ABundleAsync(bundleName)
        : new ABundle(bundleName);
    ab.Load();
    ab.AddReference();
    _loadedBundles[bundleName] = ab;
    return ab;
}
```

### ResourceManager

ResourceManager 在 BundleManager 之上提供更高层的抽象——它管理的是"资源"（Asset），而不是"Bundle"。开发者通过 ResourceManager 请求一个特定资源，而 ResourceManager 负责确定该资源在哪个 Bundle 中、加载相应的 Bundle 和依赖、从 Bundle 中提取资源、并管理资源的生命周期。

核心流程：

1. 根据资源路径查询 Manifest，确定资源所在的 Bundle
2. 调用 BundleManager 加载 Bundle（含依赖）
3. 从 Bundle 中提取资源（`LoadAsset`）
4. 缓存已加载的资源引用
5. 通过回调或事件通知调用方加载完成

**回调模式** 是 ResourceManager 的重要组成部分。对于异步加载，调用方注册一个回调函数；ResourceManager 在资源加载完成后检查状态——如果已完成则立即调用回调，否则将回调注册到待通知列表中，在未来的 `Update` 轮询中触发。

### 三种资源类型

ResourceManager 通常支持三种资源来源，按优先级降序：

**EditorResource**：仅在 Editor 模式下使用。直接通过 `AssetDatabase.LoadAssetAtPath` 从项目资源数据库加载资源，完全绕过 AssetBundle 系统。开发阶段快速迭代的首选——修改资源后立即生效，无需重新构建 Bundle。但在运行时不可用。

**Resource**：同步加载的资源类型。从 AssetBundle 中同步提取资源，阻塞调用线程直到完成。适合 Loading 阶段和确定可以快速加载的小资源。

**ResourceAsync**：异步加载的资源类型。通过协程或 `Update` 轮询检测加载状态。当异步依赖的 Bundle 也必须异步加载时，ResourceAsync 需要处理复杂的同步/异步混合依赖。

```csharp
// 同步加载
var resource = new Resource(path);
resource.Load();
var obj = resource.GetAsset<GameObject>();

// 异步加载
var asyncResource = new ResourceAsync(path);
asyncResource.Load();
// 在 Update 中检查状态
if (asyncResource.IsDone)
{
    var obj = asyncResource.GetAsset<GameObject>();
}
```

### 引用计数与延迟卸载

引用计数是资源管理的基础机制。每个 Bundle 和资源都维护一个引用计数：

- `AddReference()`：每次加载或获取资源时调用，计数 +1
- `ReduceReference()`：每次释放或不再使用时调用，计数 -1

当引用计数降为零时，资源不会立即卸载。而是被加入一个**延迟卸载列表**（`NeedUnloadList`），在 `LateUpdate` 中统一处理。延迟卸载的设计原因：

1. **防止帧内反复加载/卸载**：同一帧内可能先释放再请求同一资源，立即卸载会导致不必要的重复加载
2. **避免卸载顺序问题**：在依赖关系复杂时，批量处理可以按正确的顺序卸载（先卸载依赖方，再卸载被依赖方）
3. **降低 `Unload` 调用的频率**：`Unload(true)` 是昂贵的操作，批量处理减少调用次数

```csharp
void LateUpdate()
{
    foreach (var bundle in _needUnloadList)
    {
        bundle.UnLoad();  // 调用 AssetBundle.Unload(false)
    }
    _needUnloadList.Clear();
}
```

## 面试要点

### 打包策略

AssetBundle 的打包策略直接影响内存占用、加载速度和热更新效率。核心取舍在**打包粒度的两极**之间：

**少量大 Bundle** 的优点是文件 I/O 次数少——一次读取就能获取大量资源。缺点是内存浪费严重：即使只需要 Bundle 中的一个 Texture，也必须加载整个 Bundle 的文件镜像（虽然 LZ4 支持按需解压单个资源块，但文件镜像本身仍需完整加载）。热更新也不友好——修改一个资源需要重新下载整个大 Bundle。

**大量小 Bundle** 的优点是按需加载精准、热更新粒度细。缺点是文件读取次数多——打开和读取大量小文件的系统调用开销不容忽视。此外，如果多个小 Bundle 引用同一个依赖资源（如共享 Shader），该依赖可能被重复打包到每个 Bundle 中，造成磁盘和内存冗余。

> [!tip] 推荐策略
> 按功能模块和更新频率分层打包。Shader、字体、公共 UI 组件按类型打包到大 Bundle（低频更新）。每个功能模块的资源和场景按模块独立成 Bundle（支持按模块热更）。大型纹理和音频文件独立成 Bundle（按需加载）。

**Resources 文件夹** 的资源在应用启动时被编入红黑树索引，且永远无法被 `AssetBundle.Unload` 清理。它们要么常驻内存，要么等待 `UnloadUnusedAssets`。因此 Resources 文件夹应只存放少量真正全局需要的资源——启动画面资源、全局字体、必需的 Shader 等。

### 加载与实例化

**Asset、GameObject 与 AssetBundle 的关系** 是面试中的高频问题：

- **AssetBundle**：磁盘上的归档文件（或加载后的内存镜像），是资源的容器。一个 Bundle 包含多个资源。
- **Asset**：Bundle 中的资源数据。`Texture2D`、`Mesh`、`Material`、`GameObject`（作为 Prefab）都是 Asset。加载 Asset 意味着从 Bundle 中提取数据到原生内存。
- **GameObject**：场景中的实例。通过 `Instantiate(asset)` 从 Asset 创建。GameObject 是 Asset 的运行时副本。

**`Instantiate` = 克隆 + 引用**：Transform 和 GameObject 自身被克隆（每个实例独立）；Mesh、Texture、Material、Shader 被引用（所有实例共享）。这意味着实例化 1000 个 Prefab 不会复制 1000 份网格数据——顶点缓冲只有一份。

**完整销毁链** 必须按正确的顺序执行：

1. `Destroy(gameObject)` — 移除场景中的实例
2. `Resources.UnloadAsset(asset)` — 释放资源的原生数据（所有实例已销毁，引用计数为零）
3. `bundle.Unload(true)` — 释放 Bundle 文件镜像和所有从它加载的资源
4. `Resources.UnloadUnusedAssets()` — 全局清理所有无引用资源

跳过任何一步都可能导致内存泄漏或 Missing 引用。

### 卸载时机

**`Unload(false)` vs `Unload(true)`** 是区分经验丰富的 Unity 开发者和新手的标准问题：

- `Unload(false)`：释放 Bundle 的**文件内存镜像**，但保留已通过 `LoadAsset` 提取的资源。调用后资源仍然可用，但无法再从该 Bundle 加载新资源。推荐在 `LoadAsset` 完成后立即调用。
- `Unload(true)`：释放 Bundle 的文件内存镜像**和**所有从该 Bundle 加载的资源。无论资源是否仍被引用，都会被强制释放。场景中的引用者会变成 Missing。

**循环依赖的预防** 在构建阶段实现。构建管线使用 `HashSet<string>` 追踪已处理的 Bundle。在递归处理依赖时，如果遇到已在 `HashSet` 中的 Bundle，说明存在循环依赖——构建应当报错而非静默处理。

**LZ4 的内存复用** 是一个被低估的优势。LZ4 压缩的 Bundle 在加载时不需要额外的解压缓冲区——按需解压的单个资源块直接写入目标内存。而 LZMA 需要先将整个 Bundle 解压到一个临时缓冲区，然后才能访问资源，峰值内存是 LZ4 的约 2 倍。

**场景加载** 会自动销毁当前场景的所有 GameObject，但**不会**释放 AssetBundle 的文件内存镜像。如果场景切换时没有手动调用 `Unload`，上一场景的 Bundle 镜像会一直占用内存直到调用 `UnloadUnusedAssets` 或应用退出。因此建议在场景切换前主动卸载不再需要的 Bundle。

**Shader 的全局共享** 是 Unity 的资源管理中的一个特例。Shader 对象是全局共享的——永远不会被 `Instantiate` 复制，也永远不会因为 `Unload(true)` 而被释放（如果它仍被任何材质引用）。Shader 的 `Unload` 必须通过显式的 `Resources.UnloadAsset`。
