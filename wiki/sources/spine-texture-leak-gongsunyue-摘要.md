---
title: "Spine 立绘纹理泄漏案例 — paintinganim_gongsunyue"
type: source-summary
updated: 2026-06-17
source: "raw/gamedev/animation/spine-texture-leak-gongsunyue.md"
tags: [spine, unity, memory-leak, xlua, ondestroy]
---

# Spine 立绘纹理泄漏案例 — `paintinganim_gongsunyue`

公孙越立绘 Spine 纹理（Texture2D，约 16 MB）在对应 GameObject 销毁后仍驻留内存的根因分析与修复。本案例是 [[concepts/Spine资源管理|Spine 资源管理]] 清理三步法的**实战必要性印证**，并抽象出 [[concepts/内存泄漏调试方法论|内存泄漏调试方法论]]。

## 核心结论

纹理泄漏的**根因**是 C# `SpineUIController2.OnDestroy` 未释放 Spine 纹理——既未 `ResourceManager.UnloadAsync` 卸载 SkeletonDataAsset（引用计数不归零），也未清理 `SkeletonGraphic.canvasRenderer` 的材质/纹理绑定，更未置空 `skeletonGraphic` 字段。

> [!important] 决定性判据
> 截图前提是"GameObject 已被销毁"。Unity 中 `GameObject.Destroy` 必然触发 `OnDestroy`，而纹理仍在内存 ⇒ 唯一可能是 `OnDestroy` 没有释放纹理。这条反推锁定了根因所在。

## 关键要点

- **引用链**：`Texture2D ← SkeletonGraphic ← SpineUIController2.skeletonGraphic ← ObjectTranslator.objects ← LuaEnv`。被 xLua 钉住的 C# 包装对象沿字段引用到达纹理。
- **GO 销毁 ≠ 资源释放**：`Destroy` 只销毁原生对象，C# 托管包装置为 fake null 但仍存活；只要 `OnDestroy` 没断开纹理引用，纹理就泄漏。
- **三条遗漏路径**（旧版 OnDestroy 只清了 `skeletonDataAsset/material`）：CanvasRenderer 材质槽、ResourceManager 引用计数、`skeletonGraphic` 字段。
- **次要因素（非根因）**：xLua `ObjectTranslator` 钉住、消费者 `HeroAvatarSimple` 不主动回收 `avatarSpine`，只延长 `SpineUIController2` 短期存活；只要 `OnDestroy` 切断纹理链，钉住与否都不影响释放。
- **根因判定教训**：v2 曾把"消费者不回收"误判为治本根因，实际只改 C# `OnDestroy` 即消除泄漏。**判定根因的标准：去掉它问题是否消失。**

## 已验证修复

| 修复点 | 文件 | 内容 |
|--------|------|------|
| 清 CanvasRenderer 材质/纹理 | `SpineUIController2.cs:94-108` | `mat.mainTexture=null` + `SetMaterial(null,i)` + `SetTexture(null)` |
| 卸载 SkeletonDataAsset | `SpineUIController2.cs:110-111` | `ResourceManager.Inst.UnloadAsync(assetPath)` |
| 置空 skeletonGraphic | `SpineUIController2.cs:117` | 断开字段引用，被钉住对象不再到达纹理 |
| 清空 Lua→C# 引用 | `VisUiActor.lua:328-333` | `spineUIController`/`gameObject`/`transform` 等置 nil |

## 残留建议（非阻塞）

- `SpineUIController2.cs:113` `_skeletonDataAsset.Clear()` 补空判，避免加载失败时 NRE 跳过 `skeletonGraphic=null`。
- `VisUiActorPool.lua:125` 注释"超过10s"应为"超过60s"。

## 参见

- [[concepts/Spine资源管理|Spine 资源管理]] — 通用清理三步法（本案例是其必要性印证）
- [[concepts/内存泄漏调试方法论|内存泄漏调试方法论]] — 引用链反推根因、根因 vs 次要因素辨析
- [[concepts/XLua热补丁|XLua 热补丁]] — ObjectTranslator 对象桥
- [[concepts/Unity资源管理|Unity 资源管理]] — 引用计数与卸载
- [[concepts/CSharp内存GC|C# 内存 GC]] — 托管/原生内存、fake null
- [[sources/spine-resource-unload-摘要|Spine 资源卸载]] — 配套的清理三步法来源
