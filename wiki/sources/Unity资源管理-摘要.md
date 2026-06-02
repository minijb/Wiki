---
title: "Unity 资源管理与 AssetBundle — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/optimization/unity-asset-management.md"
tags: [unity, asset-management, assetbundle, memory]
---

# Unity 资源管理与 AssetBundle

## 来源

`raw/gamedev/optimization/unity-asset-management.md` — Unity 资源管理的完整体系：三种内存模型、资源加载/卸载管线、AssetBundle 构建与运行时架构、面试要点

## 要点

1. **三种内存类型** — 托管内存（Mono/IL2CPP GC 管理）、原生内存（Unity C++ 引擎管理 Texture/Mesh 主体数据）、非托管内存（`NativeArray<T>` 等需手动 `Dispose`）。一个 `Texture2D` 跨越托管（C# 包装器）和原生（像素数据）两块内存
2. **ScriptableObject 内存模型** — 值类型字段在托管堆上；原生资源引用（Texture/Mesh）在 C# 层只存 `m_CachedPtr` + 引用计数，实际数据在原生内存。浅拷贝 `Instantiate(so)` 复制值类型但共享原生资源；深拷贝需额外 `Instantiate(so.texture)`
3. **资源加载管线** — 磁盘 → AssetBundle 内存镜像 → 原生内存（资源数据）→ 托管内存（C# 包装器）→ 场景对象。四步层层增加内存，且释放方式各不相同
4. **`Instantiate` 语义** — `Instantiate(prefab)` 克隆 GameObject/Transform/脚本数据（托管），但引用 Mesh/Material/Texture/Shader（原生共享）。实例化 1000 个 Prefab 不会复制 1000 份网格数据
5. **AssetBundle 构建** — `BuildPipeline.BuildAssetBundles` 输出资源文件 + Manifest 文件 + 主 Bundle。压缩方式：LZMA（需全解压、峰值高）→ LZ4（按块解压、推荐）→ 无压缩（最快、文件最大）
6. **加载方法对比** — `Resources.Load` 简单但不可动态卸载；`LoadFromFile`（推荐生产）加载文件镜像；`LoadFromMemory` 额外分配解压副本；`LoadAsset` 从 Bundle 提取资源；异步用 `LoadAssetAsync<T>` 配合协程
7. **卸载策略（关键）** — 依赖方向：场景对象 → 资源实例 → Bundle 镜像 → 原生资源。`Destroy()`→`UnloadAsset()`→`Unload(true/false)`→`UnloadUnusedAssets()`
8. **`Unload(false)` vs `Unload(true)`** — `Unload(false)` 只释放 Bundle 文件镜像，保留已提取资源（推荐：加载后立即调用）；`Unload(true)` 强制释放镜像和所有资源，无视引用计数，可能导致 Missing 引用
9. **引用计数与 `UnloadUnusedAssets`** — Unity 内部为原生资源维护引用计数（场景引用 + C# 变量引用 + 资源间引用）。`UnloadUnusedAssets` 全局扫描所有资源释放计数为零者，开销大，应在 Loading/场景切换时调用
10. **依赖管理** — Bundle 间依赖自动记录但运行时加载顺序必须正确：主 Bundle → Manifest → 查询依赖 → 按序加载依赖 Bundle → 最后加载目标。**不允许循环依赖**，解决方案是提取共享资源到公共 Bundle
11. **BundleManager 模式** — 封装已加载 Bundle 缓存、依赖解析、引用计数、异步加载队列、延迟卸载列表。核心方法：`LoadInternal`（递归加载依赖 + 引用计数）、延迟卸载在 `LateUpdate` 批量处理
12. **打包策略** — 少量大 Bundle（I/O 少但内存浪费）vs 大量小 Bundle（精准但系统调用多、依赖冗余）。推荐按功能模块和更新频率分层：Shader/字体按类型打包（低频更新）、功能模块独立 Bundle（热更）、大纹理独立 Bundle（按需加载）
13. **面试要点** — AssetBundle vs Asset vs GameObject 三者关系；完整销毁链（Destroy → UnloadAsset → Unload(true) → UnloadUnusedAssets）；LZ4 内存复用无需额外解压缓冲；场景切换不自动释放 Bundle 镜像需手动 `Unload`；Shader 全局共享永不复制

## 关联 Wiki 页面

- [[concepts/Unity资源管理|Unity 资源管理]] — 概念页
- [[concepts/CSharp内存GC|C# 内存与 GC]] — 托管内存与 GC 机制
