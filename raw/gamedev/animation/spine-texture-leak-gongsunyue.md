---
title: "Spine UI 立绘纹理泄漏分析报告 — paintinganim_gongsunyue"
type: source
updated: 2026-06-17
tags: [spine, unity, memory-leak, xlua, resource-management, ondestroy]
---

# Spine UI 立绘纹理泄漏分析报告 — `paintinganim_gongsunyue`

**分析日期**：2026-06-17
**问题对象**：`paintinganim_gongsunyue`（Texture2D，公孙越立绘 Spine 纹理，约 16 MB）
**状态**：代码已修复并验证正确。本报告据已修复代码回溯真正根因。

> [!important] 决定性判据
> 截图抓取时，公孙越对应的 GameObject **已经被销毁**，但纹理仍驻留内存。Unity 中 `GameObject.Destroy` 必然触发 MonoBehaviour 的 `OnDestroy` —— 因此 `SpineUIController2.OnDestroy` **一定执行过**，而纹理仍在内存 ⇒ 唯一可能是 **`OnDestroy` 没有释放纹理**。这条判据是锁定根因的关键。

## 相关代码

| 文件 | 角色 |
|------|------|
| `Client/Assets/Script/Core/Graphic/SpineUIController2.cs` | C# Spine UI 控制器 |
| `Client/Assets/Script/Lua/Graphic/VisEntity/VisUiActor.lua` | Lua 侧 Spine UI 实体 |
| `Client/Assets/Script/Lua/Graphic/VisEntity/VisUiActorPool.lua` | Spine UI 实体对象池 |
| `Client/Assets/Script/Lua/Graphic/UISpine_new.lua` | 面板使用的 Spine 组件（持有 VisUiActor） |
| `Client/Script/Lua/Client/UI/HeroSysPanel/HeroAvatarSimple/HeroAvatarSimple.lua` | 立绘面板消费者 |
| `Client/Assets/Script/Lua/LuaGame.lua` | 驱动对象池 Update |

> [!note] 泄漏链路限定
> `HeroAvatar.lua` 走旧版 `UISpine`（`SpineUIController`），**不**走本链路。本泄漏仅限 `HeroAvatarSimple → UISpine_new → VisUiActor → SpineUIController2` 这条公孙越立绘所走的新链路。

## 1. 问题现象（引用链）

从 `11.png`、`12.png` 两张内存快照可见，`paintinganim_gongsunyue` 纹理仍被持有，引用链为：

```
Texture2D  paintinganim_gongsunyue
  ↑ mainTexture / atlas 引用
SkeletonGraphic            (Spine 运行时组件，托管对象仍存活)
  ↑ SpineUIController2.skeletonGraphic 字段
SpineUIController2         (MonoBehaviour，托管对象仍存活)
  ↑ Dictionary<object,int> 强引用
ObjectTranslator           (xLua 的 C#↔Lua 对象桥缓存)
  ↑
LuaEnv                     (Lua 虚拟机)
```

**核心事实**：GameObject 已销毁，但托管层的 `SpineUIController2` 与 `SkeletonGraphic` 两个 C# 包装对象仍被 xLua 的 `ObjectTranslator.objects` 强引用钉住，无法被 .NET GC 回收；它们内部仍持有纹理引用，导致 16 MB 纹理无法释放。

## 2. 为什么"GameObject 销毁"不等于"资源释放"

理解本泄漏的关键机制，分三层。

### 2.1 Unity 销毁 ≠ 托管对象释放

`GameObject.Destroy(go)` 只销毁引擎原生对象，并把 C# 托管包装对象（`SpineUIController2`、`SkeletonGraphic`）置为"假空"（fake null，`m_CachedPtr` 被清零，但托管对象本身仍在堆上）。**只要 .NET GC 不回收，托管对象及其字段引用就一直存在。**

### 2.2 托管 SkeletonGraphic 内部仍持有纹理

旧版 `SpineUIController2.OnDestroy` 虽然执行了：

```csharp
skeletonGraphic.skeletonDataAsset = null;
skeletonGraphic.Clear();
skeletonGraphic.material = null;
```

但这远远不够：

- `SkeletonGraphic` 内部持有运行时 `Skeleton` 对象，`Skeleton → Atlas → Texture2D`。`Clear()` 只清网格顶点，**不释放运行时 Skeleton / Atlas**。
- 纹理还绑定在 `SkeletonGraphic.canvasRenderer` 的材质上（`canvasRenderer.GetMaterial(i).mainTexture`），而上面三行**完全没碰 CanvasRenderer**。
- SkeletonData 是经 `ResourceManager.LoadAsync(assetPath, SkeletonData)` 加载的，旧版 OnDestroy **从未调用** `ResourceManager.UnloadAsync(assetPath)`，纹理的加载引用计数不归零，ResourceManager 侧也不会释放。

因此即使 material、skeletonDataAsset 字段已置空，纹理仍通过 `CanvasRenderer`、运行时 `Skeleton/Atlas`、以及 ResourceManager 引用计数三条路径被持有。

### 2.3 xLua ObjectTranslator 钉住 C# 对象

xLua 把每个被 Lua 访问过的 C# 对象以**强引用**存入 `ObjectTranslator.objects`（键即 C# 对象本身）。只要 Lua 侧还有任何引用（字段、闭包 upvalue），C# 对象就留在缓存里，.NET GC 无法回收。要释放它，必须**先断开 Lua 侧所有引用**，再等 Lua GC 触发 userdata 的 `__gc` 回调去移除缓存项。

> [!tip] 钉住 ≠ 持有纹理
> xLua 钉住 `SpineUIController2` 只是让这个**小**托管对象存活，它本身并不持有 16 MB 纹理。纹理泄漏的充要条件是"存活的 `SpineUIController2` 仍能沿字段引用到达纹理"。只要 `OnDestroy` 把这条引用链断开（置空 `skeletonGraphic`、清 CanvasRenderer、卸载 SkeletonData），钉住与否都不影响纹理释放。

## 3. 完整生命周期追踪

正常路径（**无泄漏**）：

```
HeroAvatarSimple:showSpine_Painting(...)
  → UISpine_new:setHeroSpineByCharInfo_Painting(...)
     → _getOrCreateActor → visUiActorPool:Get()        -- 从池取出 VisUiActor
     → VisUiActor:Load()                                -- 加载 uispine prefab
        → self.spineUIController = gameObject:GetComponent(SpineUIController2)  ★建立 Lua→C# 引用
        → loadSkeletonData → SpineUIController2:SetSkeletonData(加载纹理)

面板关闭/回收:
  UISpine_new:Close()/recycleActor() → visUiActorPool:Recycle(actor)  -- 回收进池(隐藏,不销毁GO)
  池内静默 → VisUiActorPool:Update() 判定超时 → ent:destroy()
     → VisUiActor:destroy() → VisEntity:destroy()
        → self:UnLoad(assetPath) → GameObject.Destroy(go)  -- ★触发 C# SpineUIController2.OnDestroy
        → self.spineUIController = nil            -- 断开 Lua→C# 引用
        → self.gameObject = nil / self.transform = nil ...
```

要点订正：

- 池的 `Update` 由 `LuaGame.lua:238` 每帧驱动；超时阈值是 **60s**（`VisUiActorPool.lua:125` `>= 60`，代码内注释"超过10s"为 stale，与实际不符）。
- 即使消费者不主动回收，面板 GameObject 被销毁时，其子节点（spine 挂在 `self._mask.gameObject` 下，见 `HeroAvatarSimple.lua:33`）会随之销毁，**`SpineUIController2.OnDestroy` 同样会执行**。也就是说，无论走池回收还是被遗弃，OnDestroy 都会跑——泄漏只取决于 OnDestroy 是否真正释放纹理。

## 4. 根因定位

### 真正根因：C# `SpineUIController2.OnDestroy` 未释放 Spine 纹理

旧版 `OnDestroy`（`SpineUIController2.cs:58-118` 区段）存在三处缺失，共同导致纹理无法释放：

1. **未通过 ResourceManager 卸载 SkeletonDataAsset**：纹理经 `SetSkeletonData` → `ResourceManager.LoadAsync(assetPath, SkeletonData)` 加载（`SpineUIController2.cs:134`），若 OnDestroy 不调用 `ResourceManager.UnloadAsync(assetPath)`，纹理的加载引用计数永不归零，ResourceManager 侧不会释放。
2. **未清理 `SkeletonGraphic.canvasRenderer` 的材质/纹理绑定**：纹理还绑在 `canvasRenderer` 的材质 `mainTexture` 上，旧版完全未触碰 `canvasRenderer`。
3. **未置空 `skeletonGraphic` 字段**：被 xLua 钉住的 `SpineUIController2` 经此字段持续指向 `SkeletonGraphic`（后者内部 `Skeleton → Atlas → Texture`）。

三层叠加 = 截图中"GO 已销毁、OnDestroy 已执行、纹理仍驻留"。

### 次要因素（放大，但非根因）：xLua 钉住 + 消费者不主动回收

- `HeroAvatarSimple.destroy/Close` 不调用 `avatarSpine:destroy()/Close()`（`HeroAvatarSimple.lua:41-43, 61-67`），`UISpine_new._visUiActor` 被遗弃（既不回池也不 destroy）。
- 遗弃的 `VisUiActor.spineUIController` 仍指向 `SpineUIController2`，xLua 把后者钉在 `ObjectTranslator.objects` 里，使其在面板关闭后短期内存活。

但这不是纹理泄漏的根因：

- GameObject 会被销毁（子节点随父节点销毁，或池 60s 超时销毁），`OnDestroy` 必然执行。
- 只要 `OnDestroy` 正确释放纹理（已修复），钉住的 `SpineUIController2` 即便存活也到达不了纹理，不影响纹理释放。
- 此外，`HeroAvatarSimple` 自身在面板销毁、从 PanelManager 移除后变为不可达，Lua GC 终将回收 `HeroAvatarSimple → avatarSpine → _visUiActor → spineUIController` 整条链，`SpineUIController2` 随之解除钉住。即 A 充其量是"及时性/风格"问题，不构成纹理泄漏。

> [!warning] 根因 vs 次要因素的辨析教训
> 早期版本（v2）曾把"消费者不回收 `avatarSpine`"（缺陷 A）误判为治本根因，并建议改 `HeroAvatarSimple.destroy/Close`。实际修复**并未碰 A**，只改了 C# 侧 `OnDestroy` + Lua 引用清理，泄漏即消除——反证根因在 C# `OnDestroy`。**判定根因的标准：去掉它问题是否消失。** 修改 A 不改 OnDestroy，泄漏仍在；只改 OnDestroy，泄漏消失 ⇒ OnDestroy 才是根因。

## 5. 代码缺陷清单（含当前状态）

| # | 文件:行 | 缺陷 | 当前状态 | 影响 |
|---|---------|------|----------|------|
| **A** | `HeroAvatarSimple.lua:41-43, 61-67` | `destroy`/`Close` 不回收 `avatarSpine` | 未改（保留） | **非纹理泄漏根因**。面板销毁后 Lua GC 终会回收整条链；`RecyclePainting()`（`168-172`）已提供显式回收入口。属及时性/风格问题。 |
| **B** | `SpineUIController2.cs:117` | `OnDestroy` 末尾未 `skeletonGraphic = null` | ✅ 已修复 | 断开 `SpineUIController2 → SkeletonGraphic` 字段引用，被钉住的 C# 对象不再到达纹理 |
| **B′** | `SpineUIController2.cs:94-108` | `OnDestroy` 未清理 `canvasRenderer` 材质/纹理（新增） | ✅ 已修复 | 逐个 `mat.mainTexture=null`、`SetMaterial(null,i)`、`SetTexture(null)`，断开 CanvasRenderer→纹理 |
| **B″** | `SpineUIController2.cs:110-111` | `OnDestroy` 未卸载 SkeletonDataAsset（新增） | ✅ 已修复 | `ResourceManager.Inst.UnloadAsync(this.assetPath)`，纹理加载引用计数归零 |
| **C** | `SpineUIController2.cs:113` | `_skeletonDataAsset.Clear()` 无空判 | ⚠️ 未改（保留） | 若 `_skeletonDataAsset==null`（加载失败/未完成即销毁）抛 NRE，跳过其后 `114-117`（含关键的 `skeletonGraphic=null`）。建议补 `if (_skeletonDataAsset != null)`。 |
| **D** | `VisUiActor.lua:321-334` | `UnLoad` 未清空 Lua 侧 C# 引用 | 🟡 部分修复 | `328-333` 已清 `spineUIController`/`gameObject`/`transform`/`rectTrans`/`parent`/`onCompletedCallback`；`325` `resMgr:UnloadAsync` 仍注释——因纹理卸载已移至 C# OnDestroy（B″），此处的共享 `uispine` prefab 句柄不卸载可接受。 |
| 池超时注释 | `VisUiActorPool.lua:125` | 注释写"超过10s"，实际阈值 `>= 60` | ⚠️ 注释 stale | 仅注释错误，逻辑无碍；建议把注释改为 60s 以免误导。 |
| **E** | `VisUiActor.lua:52-58` | `Reset` 不重置 `loadState` | 未改（保留） | **非问题**。同路径复用靠 `VisEntity:Load` 的"已 Loaded"早退（`VisEntity.lua:215`）；换路径靠 `_getOrCreateActor` 先 `Recycle` 旧 actor（`UISpine_new.lua:56-59`）。不触发泄漏。 |
| **F** | `VisUiActor.lua:84-90` / `SpineUIController2.cs:134-142` | 异步加载闭包长期持有 `self`/`this` | 未改（次要） | 加载完成前若 GO 被销毁，闭包仍被 ResourceManager 持有，延长 C# 对象存活期。非本次泄漏触发点。 |

## 6. 已应用的修复（已验证正确）

### 6.1 C# `SpineUIController2.OnDestroy` 彻底释放纹理（治本）

`SpineUIController2.cs:58-118`，关键三处：

```csharp
void OnDestroy()
{
    Debug.Log("[SpineUIController2] OnDestroy");
    if (skeletonGraphic != null)
    {
        // ... 停动画、skeletonDataAsset=null、Clear()、material=null ...

        // [B′] 清理 CanvasRenderer 的材质/纹理绑定（新增）
        var canvasRenderer = skeletonGraphic.canvasRenderer;
        if (canvasRenderer != null)
        {
            int materialCount = canvasRenderer.materialCount;
            for (int i = 0; i < materialCount; i++)
            {
                Material mat = canvasRenderer.GetMaterial(i);
                if (mat != null) mat.mainTexture = null;
                canvasRenderer.SetMaterial(null, i);
            }
            canvasRenderer.SetTexture(null);
        }
    }
    // [B″] 卸载 SkeletonDataAsset，纹理引用计数归零（新增）
    if (!string.IsNullOrEmpty(this.assetPath))
        ResourceManager.Inst.UnloadAsync(this.assetPath);
    _skeletonDataAsset.Clear();
    _skeletonDataAsset = null;
    this.defaultMaterial = null;
    this.assetPath = null;
    this.skeletonGraphic = null;          // [B] 断开对 SkeletonGraphic 的字段引用（新增）
}
```

> 这是本次泄漏的**决定性修复**：三条纹理引用路径（CanvasRenderer、ResourceManager 引用计数、`skeletonGraphic` 字段）全部切断。即便 `SpineUIController2` 仍被 xLua 钉住，也无法到达纹理。

### 6.2 Lua `VisUiActor:UnLoad` 清空 C# 引用

`VisUiActor.lua:321-334`：

```lua
function VisUiActor:UnLoad(assetPath)
    if (not IsNull(self.gameObject)) then
        GameObject.Destroy(self.gameObject)
    end
    -- resMgr:UnloadAsync(assetPath, self.handle)   -- 纹理卸载已由 C# OnDestroy 负责，此处共享 prefab 句柄不卸载
    self.loadState = Enum.LoadState.Invalid
    -- 即便绕过 destroy()，也保证断开 C# 引用
    self.spineUIController = nil
    self.gameObject = nil
    self.transform = nil
    self.rectTrans = nil
    self.parent = nil
    self.onCompletedCallback = nil
end
```

配合 `VisUiActor:destroy()`（`47-50`）已有的 `self.spineUIController = nil`，确保正常池路径下 Lua→C# 引用被断开，解除 xLua 钉住。

## 7. 残留与建议（非阻塞）

- **C**：`SpineUIController2.cs:113` `_skeletonDataAsset.Clear()` 补空判，避免加载失败时 NRE 中断、跳过 `skeletonGraphic=null`。
- **池超时注释**：`VisUiActorPool.lua:125` 注释"超过10s"改为"超过60s"。
- **A（可选）**：若希望面板关闭后更及时地回收 actor 到池（减少短期驻留），可在 `HeroAvatarSimple:Close` 调 `self.avatarSpine:Close()`；但这是资源调度优化，与纹理泄漏无关。

## 8. 验证方法

1. **重抓内存快照**，搜索 `paintinganim_gongsunyue`：
   - 期望：关闭公孙越立绘面板后，Texture2D 的 `Referenced By` 不再出现 `SpineUIController2` / `XLua.ObjectTranslator`，纹理被释放。
   - 若仍出现，检查是否还有其它 Lua 闭包/字段持有 `VisUiActor`，或 `OnDestroy` 是否因 C（NRE）提前中断。
2. **复现场景**：打开含公孙越立绘的面板 → 关闭面板 → 抓快照。
3. **观察 xLua 缓存**：确认 `XLua.ObjectTranslator` 下 `SpineUIController2` 残留实例在 Lua GC 后清除（A 路径下为延迟回收，非永久泄漏）。
4. **断点/日志确认**：`SpineUIController2.OnDestroy` 中 `skeletonGraphic = null`、`canvasRenderer.SetTexture(null)`、`ResourceManager.UnloadAsync` 均执行；留意 `_skeletonDataAsset.Clear()` 是否因空判缺失抛 NRE。

## 9. 结论

`paintinganim_gongsunyue` 纹理泄漏的**根因**是 C# `SpineUIController2.OnDestroy` 未释放 Spine 纹理资源——它既未通过 `ResourceManager.UnloadAsync` 卸载 SkeletonDataAsset（纹理引用计数不归零），也未清理 `SkeletonGraphic.canvasRenderer` 的材质/纹理绑定，更未置空 `skeletonGraphic` 字段。截图前提"GameObject 已销毁"意味着 `OnDestroy` 必然执行过，而纹理仍驻留，唯一解释就是 `OnDestroy` 没有释放纹理。

xLua `ObjectTranslator` 钉住 `SpineUIController2`、`HeroAvatarSimple` 不主动回收 `avatarSpine`，是让 `SpineUIController2` 短期存活的**次要因素**，但只要 `OnDestroy` 切断纹理引用链，钉住与否都不影响纹理释放——故并非根因。

**修复落点**：

1. `SpineUIController2.OnDestroy`：清 CanvasRenderer 材质/纹理 + `ResourceManager.UnloadAsync` 卸载 SkeletonData + `skeletonGraphic = null`（治本，已完成）。
2. `VisUiActor:UnLoad`：清空 `spineUIController` 等 Lua→C# 引用（已完成）。
3. 残留：`_skeletonDataAsset.Clear()` 补空判（建议）。

## 参见

- [[concepts/Spine资源管理|Spine 资源管理]] — 通用清理三步法（本案例是其必要性印证）
- [[concepts/内存泄漏调试方法论|内存泄漏调试方法论]] — 本案例抽象出的根因定位方法
- [[concepts/XLua热补丁|XLua 热补丁]] — ObjectTranslator 对象桥机制
- [[concepts/Unity资源管理|Unity 资源管理]] — ResourceManager 引用计数
- [[concepts/CSharp内存GC|C# 内存 GC]] — 托管/原生内存、fake null
