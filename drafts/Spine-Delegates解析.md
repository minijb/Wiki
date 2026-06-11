---
status: archived
archived-to: raw/gamedev/animation/spine-delegates.md
updated: 2026-06-11
tags: [spine, unity, delegate, callback, skeleton-animation, skeleton-graphic]
---

# Spine Delegates 完全解析

Spine Unity 运行时暴露了多组 delegate（C# event / `Action`）用于在动画管线的不同阶段插入自定义逻辑。理解每个 delegate 的触发时机、参数和使用场景是正确使用 Spine 的前提。

---

## 分类总览

Spine 的 delegate 按所属对象分为四组：

| 分组 | 所属对象 | 声明类型 | 数量 | SkeletonAnimation | SkeletonGraphic |
|:-----|:-----|:-----|:-----|:-----|:-----|
| 更新阶段回调 | `SkeletonAnimation` | `UpdateBonesDelegate` | 4 | 全部支持 | **不支持** |
| 渲染阶段回调 | `SkeletonRenderer` | `Action<SkeletonRenderer>` | 2 | 全部支持 | 全部支持 |
| 动画状态事件 | `AnimationState` | C# event | 6 | 全部支持 | 全部支持 |
| 轨道条目事件 | `TrackEntry` | C# event | 6 | 全部支持 | 全部支持 |

---

## Unity 帧循环中的 delegate 触发时序

```
MonoBehaviour.Update()
  └─ [用户脚本 Update]
  └─ SkeletonAnimation.Update()
       ├─ BeforeApply         ← 动画应用到骨骼之前
       ├─ [动画应用到骨骼局部值]
       ├─ UpdateLocal         ← 局部值已写入，世界值未计算
       ├─ [skeleton.UpdateWorldTransform() — 计算世界变换]
       ├─ UpdateComplete      ← 世界值只读
       ├─ UpdateWorld         ← 修改局部值，触发第二次 UpdateWorldTransform
       └─ [skeleton.UpdateWorldTransform() — 第二次，仅当订阅了 UpdateWorld]

MonoBehaviour.LateUpdate()
  └─ [用户脚本 LateUpdate]
  └─ SkeletonRenderer.LateUpdate()
       ├─ [更新 mesh 顶点数据]
       ├─ [赋值 Material 到 Renderer]
       └─ OnMeshAndMaterialsUpdated   ← 每帧最后一站

Canvas 渲染管线（仅 SkeletonGraphic）
  └─ Canvas.SendWillRenderCanvases()
       └─ SkeletonGraphic 通过 CanvasRenderer 提交渲染数据
```

> [!note] SkeletonGraphic 的 `OnMeshAndMaterialsUpdated` 触发时机
> `SkeletonGraphic` 继承自 `MaskableGraphic`（而非 `SkeletonRenderer`），但内部的 mesh 更新和 `OnMeshAndMaterialsUpdated` 回调同样在 `LateUpdate` 阶段触发。两者时序一致，区别在于 `SkeletonGraphic` 使用 `CanvasRenderer` 且无更新阶段回调。

---

## 一、更新阶段回调（仅 SkeletonAnimation）

这四个 delegate 是 `SkeletonAnimation` 上的 C# event，签名统一为 `delegate void UpdateBonesDelegate(ISkeletonAnimation animated)`。它们在每帧 `Update()` 的不同阶段依次触发。

> [!warning] 仅 SkeletonAnimation 支持
> `SkeletonGraphic` **不暴露**这四个回调。如需在 UI Spine 上做骨骼级操作，改用 `SkeletonAnimation` 或通过 `AnimationState` 事件间接处理。

### `BeforeApply`

- **触发时机**：`Update()` 中动画数据应用到 `Skeleton` 之前
- **典型用途**：为"不会被当前动画驱动的骨骼"设置兜底值；根据外部输入预设置骨骼状态
- **注意**：修改的骨骼局部值会被后续动画覆盖（如果该骨骼被当前动画驱动的话）。因此 `BeforeApply` 适合设置**动画未覆盖的骨骼**，而非修改动画正在控制的值。

```csharp
// 适用：SkeletonAnimation
// 场景：武器骨骼不受动画驱动，由代码根据鼠标位置控制
skeletonAnimation.BeforeApply += (anim) => {
    var weaponBone = anim.Skeleton.FindBone("weapon_aim");
    // 该骨骼不在动画中，BeforeApply 设置后不会被覆盖
    weaponBone.Rotation = aimAngle;
};
```

### `UpdateLocal`

- **触发时机**：动画应用到骨骼局部值之后，世界变换计算之前
- **骨骼状态**：局部值（`X`, `Y`, `Rotation`, `ScaleX`, `ScaleY`）已是最新
- **典型用途**：读取动画后的局部值；对局部值施加偏移（不会被动画覆盖，因为动画已应用完毕）
- **注意**：世界值（`WorldX`, `WorldY` 等）仍是上一帧的旧值

```csharp
// 适用：SkeletonAnimation
// 场景：在动画基础上叠加额外的旋转偏移（如呼吸导致的武器抖动）
skeletonAnimation.UpdateLocal += (anim) => {
    var bone = anim.Skeleton.FindBone("weapon_hand");
    bone.Rotation += Random.Range(-1f, 1f); // 轻微抖动
};
```

### `UpdateComplete`

- **触发时机**：`skeleton.UpdateWorldTransform()` 完成后，所有骨骼世界值已计算
- **骨骼状态**：世界值（`WorldX`, `WorldY`, `WorldRotation` 等）已是最新
- **典型用途**：读取世界坐标挂接特效/UI/碰撞盒
- **关键限制**：**只读阶段**——修改局部值不会影响当前帧的世界值（会等到下一帧 `Update()` 才生效）

```csharp
// 适用：SkeletonAnimation
// 场景：将世界坐标传递给特效系统
skeletonAnimation.UpdateComplete += (anim) => {
    var headBone = anim.Skeleton.FindBone("head");
    // 世界值已结算，可以安全读取
    hpBar.position = new Vector3(headBone.WorldX, headBone.WorldY, 0);
};
```

### `UpdateWorld`

- **触发时机**：`UpdateComplete` 之后
- **性能代价**：如果订阅了此事件，运行时将**第二次调用 `skeleton.UpdateWorldTransform()`**。复杂 skeleton 上此开销不可忽视
- **典型用途**：实现**自定义约束**——根据世界值反推修改局部值，第二次 `UpdateWorldTransform` 会让修改在当前帧立即生效
- **与 `UpdateComplete` 的区别**：`UpdateComplete` 只能读；`UpdateWorld` 可以在修改局部值后触发第二次世界值重算

```csharp
// 适用：SkeletonAnimation
// 场景：让武器骨骼指向鼠标位置（自定义 IK 约束）
skeletonAnimation.UpdateWorld += (anim) => {
    var tip = anim.Skeleton.FindBone("weapon_tip");
    Vector2 toTarget = mouseWorldPos - new Vector2(tip.WorldX, tip.WorldY);
    tip.Rotation = Mathf.Atan2(toTarget.y, toTarget.x) * Mathf.Rad2Deg;
    // 第二次 UpdateWorldTransform 会让修改在当前帧显示
};
```

> [!tip] 官方推荐的订阅模式
> 先 `-=` 再 `+=` 防止重复订阅（对象池复用或多次初始化场景）：
> ```csharp
> skeletonAnimation.UpdateComplete -= AfterUpdateComplete;
> skeletonAnimation.UpdateComplete += AfterUpdateComplete;
> ```

### 执行顺序控制

使用 `[DefaultExecutionOrder(-1)]` 让自己的 `MonoBehaviour.Update()` 在 SkeletonAnimation 之前运行：

```csharp
// 适用：SkeletonAnimation
[DefaultExecutionOrder(-1)]
public class PreSpineSetup : MonoBehaviour {
    void Update() {
        // 在 SkeletonAnimation.Update() 之前执行
    }
}
```

---

## 二、渲染阶段回调（SkeletonAnimation + SkeletonGraphic）

`SkeletonRenderer` 基类提供两个渲染管线 delegate，签名均为 `Action<SkeletonRenderer>`。**SkeletonAnimation 和 SkeletonGraphic 均支持。**

### `OnRebuild`

- **触发时机**：Skeleton 数据加载完成、mesh 首次构建之后（`Initialize(true)` 内部）
- **触发条件**：首次设置 `skeletonDataAsset`、运行时 `Initialize(true)` 重建、SkeletonDataAsset 替换
- **频率**：**一次性**，仅在 skeleton 完全重建时
- **典型用途**：初始化时添加自定义材质、缓存 skeleton 引用、记录初始状态

```csharp
// 适用：SkeletonAnimation / SkeletonGraphic
skeletonAnimation.OnRebuild += (renderer) => {
    // Skeleton 重建后，在 MeshRenderer 上设置自定义材质
    var mr = renderer.GetComponent<MeshRenderer>();
    mr.sharedMaterial = customOutlineMaterial;  // 用 sharedMaterial 避免创建实例
};
```

### `OnMeshAndMaterialsUpdated`

- **触发时机**：**每帧 `LateUpdate()` 结束时**，mesh 顶点更新完毕、所有 Material 已赋值到 Renderer 后
- **频率**：**每帧**
- **参数**：`SkeletonRenderer` — 触发回调的渲染器实例
- **典型用途**：
  - 动态修改 Material 属性（灰度、溶解、外描边等）
  - 动态装配后处理 mesh
  - 性能分析：记录每帧顶点数/Draw Call

> [!warning] 订阅/取消订阅生命周期
> 此 delegate 每帧触发，必须正确管理订阅：
> ```csharp
> // 订阅
> skeletonGraphic.OnMeshAndMaterialsUpdated += HandleRebuild;
> // 取消订阅 — 在 OnDisable / OnDestroy / Dispose 中执行
> skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;
> ```
> 不取消会导致：内存泄露（GameObject 销毁后 delegate 仍持有目标对象引用）、重复调用（多次订阅同一方法）。

```csharp
// 适用：SkeletonGraphic
// 场景：运行时灰度化 — 在图形组件上修改材质属性
void OnEnable() {
    skeletonGraphic.OnMeshAndMaterialsUpdated += ApplyGrayScale;
}
void OnDisable() {
    skeletonGraphic.OnMeshAndMaterialsUpdated -= ApplyGrayScale;
}
void ApplyGrayScale(SkeletonRenderer renderer) {
    // SkeletonGraphic 使用 CanvasRenderer，material 属性通过 graphic 自身访问
    skeletonGraphic.material.SetFloat("_GrayScale", isGray ? 1 : 0);
}
```

```csharp
// 适用：SkeletonAnimation
// 场景：运行时的自定义渲染后处理
void OnEnable() {
    skeletonAnimation.OnMeshAndMaterialsUpdated += ApplyCustomMaterial;
}
void OnDisable() {
    skeletonAnimation.OnMeshAndMaterialsUpdated -= ApplyCustomMaterial;
}
void ApplyCustomMaterial(SkeletonRenderer renderer) {
    var mr = renderer.GetComponent<MeshRenderer>();
    // 注意：.material 每帧会创建新实例 → GC 压力
    // 若只需读取/设置 shader 属性，可改用 MaterialPropertyBlock 或 SetPropertyBlock
    mr.GetPropertyBlock(_mpb);  // _mpb 缓存复用
    _mpb.SetFloat(Shader.PropertyToID("_FillPhase"), fillAmount);
    mr.SetPropertyBlock(_mpb);
}
```

### SkeletonGraphic 与 SkeletonAnimation 的 delegate 差异

| 特性 | `SkeletonAnimation` | `SkeletonGraphic` |
|:-----|:-----|:-----|
| 渲染器类型 | `MeshRenderer` | `CanvasRenderer` |
| 更新阶段回调 | `BeforeApply` 等 4 个 | **无** |
| `OnRebuild` / `OnMeshAndMaterialsUpdated` | 支持 | 支持 |
| `AnimationState` 事件 | 支持 | 支持 |
| Material 替换方式 | `CustomMaterialOverride` / `MaterialPropertyBlock` | `CustomTextureOverride` / `CustomMaterialOverride` |
| 推荐场景 | 游戏内角色/物体 | UI 中的 Spine 立绘/动画 |

> [!note] SkeletonGraphic 的 Canvas 渲染周期
> `SkeletonGraphic` 继承自 `MaskableGraphic`（Unity UI 体系），mesh 数据通过 `CanvasRenderer` 提交。其 `OnMeshAndMaterialsUpdated` 虽在 `LateUpdate` 触发，但实际渲染由 Canvas 系统的 `WillRenderCanvases` 回调驱动。对使用者而言，接口和行为与 `SkeletonAnimation` 一致。

---

## 三、动画状态事件（AnimationState）

`AnimationState` 提供六个 C# event，是**全局级别**的事件——监听**所有轨道**上的动画状态变化。

| 事件 | 触发时机 | 参数 |
|:-----|:-----|:-----|
| `Start` | 动画开始播放 | `TrackEntry` |
| `Interrupt` | 动画被中断（新动画替换 / `ClearTrack`） | `TrackEntry` |
| `Complete` | 动画完整播放一次（循环动画每循环触发） | `TrackEntry` |
| `End` | 动画播放结束 | `TrackEntry` |
| `Dispose` | TrackEntry 被销毁，动画从轨道移除 | `TrackEntry` |
| `Event` | Spine 编辑器中设置的自定义事件触发 | `TrackEntry`, `Spine.Event` |

**触发顺序**：TrackEntry 上的事件**先于** AnimationState 上的同名事件触发（后面会讲 TrackEntry）。

### 事件触发链

非循环动画（`loop = false`）：
```
SetAnimation(0, "attack", false)
  → Start（TrackEntry 先 → AnimationState 后）
  → [播放中... 可能触发 Event]
  → Complete
  → End
  → [TrackEntry 被替换或 ClearTrack]
  → Dispose
```

循环动画（`loop = true`）：
```
SetAnimation(0, "idle", true)
  → Start
  → Complete（第一个循环结束）
  → Complete（第二个循环结束）
  → ...（每循环一次）
  → Interrupt（被新动画中断）
  → End
  → Dispose
```

> [!warning] Interrupt 时不触发 Complete
> 当一个动画被新动画中断时，**不会触发 `Complete`**，而是触发 `Interrupt` → `End`。

### 常用模式

```csharp
// 全局监听 — 适用于 AnimationState（SkeletonAnimation / SkeletonGraphic 均可）
void Start() {
    skeletonAnimation.AnimationState.Start += OnAnyAnimationStart;
    skeletonAnimation.AnimationState.Complete += OnAnyAnimationComplete;
    skeletonAnimation.AnimationState.Event += OnUserDefinedEvent;
}

void OnAnyAnimationStart(TrackEntry entry) {
    Debug.Log($"Track {entry.TrackIndex}: animation started");
}
```

```csharp
// 性能关键：缓存 EventData 引用，引用比较零 GC
// 适用于 AnimationState（SkeletonAnimation / SkeletonGraphic 均可）
Spine.EventData footstepEvent;

void Awake() {
    // SkeletonGraphic 同样通过 .Skeleton.Data 访问
    footstepEvent = skeletonAnimation.Skeleton.Data.FindEvent("footstep");
    skeletonAnimation.AnimationState.Event += HandleEvent;
}

void HandleEvent(TrackEntry entry, Spine.Event e) {
    if (e.Data == footstepEvent) {  // 引用比较，零 GC 开销
        AudioSource.PlayClipAtPoint(footstepSFX, transform.position);
    }
}
```

---

## 四、轨道条目事件（TrackEntry）

`TrackEntry` 是 `SetAnimation` / `AddAnimation` 的返回值，提供与 AnimationState 相同的六种事件，但**仅对该次播放实例生效**。

```csharp
// 适用：AnimationState（SkeletonAnimation / SkeletonGraphic 均可）
// 单实例监听 — 精确控制某次播放
TrackEntry attack = skeletonAnimation.AnimationState.SetAnimation(0, "attack", false);
attack.Complete += OnAttackComplete;

void OnAttackComplete(TrackEntry entry) {
    skeletonAnimation.AnimationState.SetAnimation(0, "idle", true);
}
```

> [!warning] TrackEntry 生命周期
> 收到 `Dispose` 事件后，`TrackEntry` 对象失效。不应再存储或访问该引用。

### 全局 vs 单实例：选择指南

| 场景 | 推荐 | 理由 |
|:-----|:-----|:-----|
| 所有动画的通用逻辑（UI 更新、状态机） | AnimationState 全局 | 一次订阅覆盖所有 |
| 特定某次播放（技能回调、连招衔接） | TrackEntry 单实例 | 精确到本次，互不干扰 |
| 性能敏感（全局需过滤大量不相关事件） | TrackEntry 单实例 | 免去过滤开销 |
| 音频/特效等自定义事件 | 两者皆可 | 单实例更精确 |

---

## 五、使用场景速查表

| 需求 | delegate | 适用范围 | 理由 | **常见误用** |
|:-----|:-----|:-----|:-----|:-----|
| 为动画不驱动的骨骼设初始值 | `BeforeApply` | SA | 动画应用前最后机会 | 试图修改动画驱动的骨骼值（会被覆盖） |
| 在动画基础上叠加偏移 | `UpdateLocal` | SA | 动画已应用，局部值可安全修改 | 读取世界值（此时仍是旧值） |
| 获取骨骼世界坐标 | `UpdateComplete` | SA | 世界值已结算，只读安全 | 修改局部值（当前帧不生效） |
| 自定义骨骼约束（IK/瞄准） | `UpdateWorld` | SA | 修改后触发第二次重算 | 忽略性能代价（第二次 UpdateWorldTransform） |
| Skeleton 初始化一次性配置 | `OnRebuild` | SA + SG | 仅在重建时触发，低成本 | 当成每帧回调使用 |
| 每帧动态修改材质 | `OnMeshAndMaterialsUpdated` | SA + SG | LateUpdate 最后一站 | 每帧 `GetComponent` / `new MaterialPropertyBlock` |
| 动态换装后处理 | `OnMeshAndMaterialsUpdated` | SA + SG | mesh 已是最新 | 未取消订阅导致泄露 |
| 监听所有动画状态 | AnimationState 事件 | SA + SG | 一次订阅全局 | 忘记 `-=` 导致重复触发 |
| 技能结束切回 Idle | TrackEntry `Complete` | SA + SG | 精确到本次播放 | 在 Dispose 后使用 TrackEntry |
| 脚步声/特效触发 | `Event`（引用比较） | SA + SG | 零 GC | 用字符串比较而非引用比较 |

> SA = SkeletonAnimation，SG = SkeletonGraphic

---

## 六、实战模式

### 模式 1：订阅-取消订阅对（防泄露）

```csharp
// 适用于所有 delegate，OnMeshAndMaterialsUpdated 尤为重要
void OnEnable()  => sg.OnMeshAndMaterialsUpdated += Handler;
void OnDisable() => sg.OnMeshAndMaterialsUpdated -= Handler;
```

项目中的实际案例见 [[Spine资源卸载指南]] 和 [[SpineFixIterations|修复迭代记录]]——`Dispose()` 中取消 `OnMeshAndMaterialsUpdated` 订阅是防止资源泄露的关键步骤。

### 模式 2：一次性初始化（OnRebuild）

```csharp
// 仅在 skeleton 首次构建时执行
bool _initialized;
void OnEnable() {
    sa.OnRebuild += OnSkeletonReady;
}
void OnSkeletonReady(SkeletonRenderer r) {
    if (_initialized) return;
    _initialized = true;
    // 一次性配置...
    sa.OnRebuild -= OnSkeletonReady; // 完成后取消
}
```

### 模式 3：骨骼世界坐标采集（UpdateComplete）

```csharp
// 缓存骨骼引用 + 在 UpdateComplete 只读世界值
Bone _headBone;
void Awake() => _headBone = sa.Skeleton.FindBone("head");
void OnEnable() => sa.UpdateComplete += SyncWorldPos;
void SyncWorldPos(ISkeletonAnimation anim) {
    // 缓存了 Bone 引用，无需每帧 FindBone
    _target.position = new Vector3(_headBone.WorldX, _headBone.WorldY, 0);
}
```

---

## 七、常见陷阱

### 1. 未取消订阅导致内存泄露

**原因**：`Action<SkeletonRenderer>` 的订阅形成 `SkeletonRenderer` → delegate → 目标对象 的强引用链。GameObject Destroy 后不取消订阅，目标对象无法 GC。

**修复**：在 `OnDisable` / `OnDestroy` / `Dispose` 中 `-=` 取消。

```csharp
// ❌ 泄露
skeletonGraphic.OnMeshAndMaterialsUpdated += HandleRebuild;
Destroy(gameObject); // HandleRebuild 所在对象仍被引用

// ✅ 正确
void OnDestroy() {
    skeletonGraphic.OnMeshAndMaterialsUpdated -= HandleRebuild;
}
```

### 2. `UpdateComplete` 中修改骨骼值

`UpdateComplete` 全部世界值已结算，此时改局部值：当前帧不生效，下一帧才生效。若需当前帧生效，用 `UpdateWorld`（但有第二次 `UpdateWorldTransform` 的性能代价）。

### 3. TrackEntry 在 Dispose 后使用

`TrackEntry` 在动画被移除后失效。动画切换或 `ClearTrack` 触发 Dispose，之后访问会不可预期。

### 4. 在 SkeletonGraphic 上使用更新阶段回调

`SkeletonGraphic` 不暴露 `BeforeApply` / `UpdateLocal` / `UpdateComplete` / `UpdateWorld`。替代方案：
- 改用 `SkeletonAnimation`（MeshRenderer 模式）
- 通过 `AnimationState` 事件间接处理

### 5. `OnMeshAndMaterialsUpdated` 回调中的性能陷阱

每帧触发，回调中避免：
- `GetComponent<MeshRenderer>()` → 缓存引用
- `Shader.PropertyToID("name")` → 静态缓存 `int`
- `new MaterialPropertyBlock()` → 复用实例
- `FindBone("name")` → 骨骼不会运行时增删，启动时缓存
- `.material`（创建实例副本）→ 改用 `.sharedMaterial` 或 `MaterialPropertyBlock`

### 6. `UpdateWorld` 的隐形成本

官方文档明确指出：订阅 `UpdateWorld` 会触发**第二次 `skeleton.UpdateWorldTransform()`**。在复杂 skeleton（多骨骼、多约束）上这是一个非平凡的 CPU 开销。仅在确实需要自定义约束时才订阅。

---

## 参见

- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — Spine 基础概念与 API
- [[sources/Unity动画与Spine-摘要|Spine 来源摘要]]
- [[Spine资源卸载指南]] — delegate 在实际 Dispose 流程中的使用
- [[SpineFixIterations|修复迭代记录]] — `OnMeshAndMaterialsUpdated` 的泄漏修复迭代
- [[concepts/Unity性能优化|Unity 性能优化]] — delegate 相关的 GC/内存优化策略
