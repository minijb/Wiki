---
title: Unity 动画系统与 Spine
type: source
updated: 2026-06-02
tags:
  - unity
  - animation
  - spine
  - animator
---

# Unity 动画系统与 Spine

## Unity 动画基础

Unity 提供四类核心动画资源：**Animation Clip**（动画片段）、**Animator Controller**（动画状态机）、**Animation 组件**（旧版）和 **Avatar**（人形动画替身）。现代 Unity 项目以 Animator Controller + Animation Clip + Avatar 为主要工作流。

### Animation Clip

Animation Clip 是 Unity 中最基础的动画数据单元，可直接在 Animation 窗口中创建和编辑。

**核心特性：**

- 可以控制 GameObject 上任意组件（Component）的属性变换，包括 `Transform` 的位置/旋转/缩放、材质颜色、UI 元素属性等
- 在 Animation 窗口中点击 **Curves** 视图可以查看和编辑动画曲线，精确控制关键帧之间的插值行为
- 动画曲线支持多种插值模式：Constant、Linear、Freeform（Bezier 曲线）

**动画复用机制：**

> [!note] 复用原理
> 只要目标 GameObject 上存在对应的组件（Component Type），Animation Clip 就可以在不同对象间复用。这是 Unity 动画系统的核心设计思路。

**多对象处理（Path-based Matching）：**

当 Animation Clip 需要控制多个子物体时，Unity 使用 **路径匹配（Path Matching）** 机制。每个被动画控制的物体在 Clip 中存储一个相对路径（Path），对应 GameObject 层级中的子物体名称。只要子物体的名称和层级路径相同，动画就可以复用。

这正是 Humanoid 动画跨模型复用的底层原理：骨骼文件（Avatar）记录骨骼映射关系，Animation Clip 只存储发生变化的骨骼变换数据。

### Root Motion

Root Motion 控制角色动画是否驱动 GameObject 的世界空间位移。

| 模式 | 行为 |
|:-----|:-----|
| **关闭 Root Motion** | 动画直接修改 GameObject 每一帧的绝对位置和旋转值 |
| **开启 Root Motion** | 使用动画的相对位移（deltaPosition）和相对旋转（deltaRotation），通过插值驱动移动 |

> [!warning] 关键差异
> 关闭时动画数据覆盖世界坐标（绝对值模式），角色位置完全由动画曲线决定。开启时使用增量位移，配合物理和脚本逻辑实现自然移动，不受动画绝对位置约束。

### Avatar 与动画复用

Avatar 是 Unity 的骨骼映射系统，解决**动画跨模型复用**问题。它建立骨骼到肌肉（Muscle）的映射，使不同骨骼结构的模型可以共享同一套动画。

**工作原理：**

1. 将模型 A 的骨骼结构通过 **AAvatar** 映射为标准化的人形骨骼
2. 将模型 B 的骨骼结构通过 **BAvatar** 映射为同一套标准化骨骼
3. 动画数据通过 Avatar 的肌肉拉伸变换（Muscle-based Stretching）来驱动不同体型

**Avatar 创建流程：**

```
Model → Rig → Animation Type → Avatar Definition → Configure
```

1. 选中模型资源，切换到 **Rig** 选项卡
2. 设置 **Animation Type**（Humanoid / Generic）
3. 在 **Avatar Definition** 中点击 `Create From This Model` 创建 Avatar
4. 点击 **Configure** 进入骨骼配置界面

**骨骼配置规则：**

- **实线骨骼**：必选（Required），Humanoid 必须映射的骨骼
- **虚线骨骼**：可选（Optional），缺失不影响动画复用

**复用方式：**

在 GameObject 的 Animator 组件上设置对应的 Avatar 即可实现动画复用。Animator 组件通过 Avatar 将标准化的动画数据映射回目标模型的骨骼层级。

---

## Animator 状态机

Animator 组件是 Unity 动画系统的核心控制器组件，作为根节点（Root Component）挂载在 GameObject 上。动画状态机（Animator Controller）通过层级路径匹配（Hierarchy-based Path Matching）控制子物体的动画属性——**Avatar 是例外**，它通过骨骼映射而非路径匹配。

### Animator 组件核心属性

**Apply Root Motion：** 是否使用动画自带的位移数据驱动 GameObject 的世界空间移动。

**Update Mode（更新模式）：**

| 模式 | 适用场景 |
|:-----|:-----|
| **Normal** | 与 `Update` 同步更新，适用于大多数游戏 |
| **Animate Physics** | 与 `FixedUpdate` 同步，适用于物理驱动的角色（如 ACT 动作游戏） |
| **Unscaled Time** | 不受 `Time.timeScale` 影响，适用于 UI 动画或暂停菜单 |

**Culling Mode（剔除模式）：**

| 模式 | 行为 |
|:-----|:-----|
| **Always Animate** | 无论是否可见都更新 |
| **Cull Update Transforms** | 不可见时跳过 Transform 更新，但保留状态机逻辑 |
| **Cull Completely** | 不可见时完全停止动画更新 |

### 状态与参数

Animator Controller 使用 **Layer** 和 **Parameters** 组织动画逻辑。

**Layer（动画层）：**
用于组合不同身体部位的动画（如上半身射击 + 下半身行走），通过权重（Weight）和遮罩（Avatar Mask）控制混合比例。

**Parameters（参数）：**

| 参数类型 | 用途 |
|:-----|:-----|
| **Float** | 连续值控制，如移动速度、混合权重 |
| **Int** | 离散状态控制，如装备索引、技能等级 |
| **Bool** | 开关状态，如是否在地面、是否持武 |
| **Trigger** | 瞬时触发，如攻击、跳跃（自动复位） |

> [!tip] 性能优化
> 使用 `Animator.StringToHash("参数名")` 获取 int 类型的参数 Hash 值，再用 `animator.SetFloat(hash, value)` 设置，避免每次字符串查找的 GC 开销。

**三种动画状态类型：**

1. **单 Clip 状态（Animation Clip）**：播放单个动画片段
2. **BlendTree 状态（混合树）**：通过参数控制多个动画片段之间的混合
3. **子状态机（Sub-State Machine）**：嵌套另一个 Animator Controller 作为状态

**状态属性详解：**

| 属性 | 说明 |
|:-----|:-----|
| **Speed** | 动画播放速度（默认 1.0） |
| **Multiplier** | 速度乘数，可绑定 Parameter 在代码中动态修改（实际速度 = Speed × Multiplier） |
| **Motion Time** | 播放动画的特定时间点，范围 `0`~`1`（归一化时间），可用于从指定帧开始播放 |
| **Mirror** | 镜像动画（仅对 Humanoid Avatar 有效），将动画左右翻转 |
| **Cycle Offset** | 循环播放的起始偏移量（不切割动画，只改变循环起点） |
| **Foot IK** | 人形 Avatar 的脚部 IK 绑定，通过 `animator.SetIKPosition()` 和 `animator.SetIKPositionWeight()` 控制 |
| **Write Defaults** | 对动画未修改的属性是否写入默认值。本意用于**对象池**管理：开启时每次播放动画会将未修改属性恢复为 `OnEnable` 时的默认值 |

**常用代码接口：**

```csharp
// 获取当前动画状态信息
AnimatorStateInfo stateInfo = animator.GetCurrentAnimatorStateInfo(layerIndex);
// 判断是否在指定 Tag 的状态中
bool isPlaying = stateInfo.IsTag("Attack");

// Hash 化参数访问
int speedHash = Animator.StringToHash("Speed");
animator.SetFloat(speedHash, 1.5f);
```

### 动画转换（Transitions）

状态之间的转换定义了动画切换的条件和方式。

**核心属性：**

| 属性 | 说明 |
|:-----|:-----|
| **Has Exit Time** | 是否等待当前动画播放到一定完成度再切换 |
| **Transition Duration** | 过渡动画的持续时长 |
| **Transition Offset** | 目标动画的起始位置偏移 |
| **Interruption Source** | 转换能否被其他转换打断 |
| **Conditions** | 触发转换的参数条件 |

**Has Exit Time 详解：**

- **开启**：需等待当前动画播放到指定时间（>Exit Time 阈值）后才能切换，保证动画的完整性
- **关闭**：无退出等待时间，条件满足时立即触发下一动画

**Interruption Source（打断源）：**

| 选项 | 行为 |
|:-----|:-----|
| **None** | 不允许任何打断 |
| **Current** | 可以被当前状态的其他转换打断（`Ordered Interruption` 勾选时只有更高优先级的转换才可打断） |
| **Next** | 可以被目标状态的转换打断 |
| **Current Then Next** | 优先判断当前状态转换，再判断目标状态转换 |
| **Next Then Current** | 优先判断目标状态转换，再判断当前状态转换 |

**转换优先级控制：**

- **Solo**：该转换独占当前状态——同一路径存在多个转换时，只有标记 Solo 的转换被考虑
- **Mute**：该转换被禁用——即使条件满足也永不触发

### BlendTree（混合树）

BlendTree 使用一个或多个参数控制一组动画片段之间的平滑混合。

**常见混合类型：**

| 类型 | 用途 |
|:-----|:-----|
| **1D** | 单个参数控制，如速度 0→1 对应 Idle→Walk→Run |
| **2D Simple Directional** | 两个参数，方向控制，如移动方向（前后左右） |
| **2D Freeform Directional** | 两个参数，自由方向混合 |
| **2D Freeform Cartesian** | 两个独立参数，如速度和转向角 |
| **Direct** | 直接控制每个动画的权重 |

BlendTree 内部可以嵌套子 BlendTree，实现多维度动画混合（如速度 + 方向 + 坡度）。

### Root Motion 配置

Root Motion 适用于 **Generic Avatar** 和 **Humanoid Avatar**。在 Generic 中需先创建 Avatar 并指定 **Root Node**，然后将角色根骨骼的绝对坐标和角度应用到 GameObject 上。

**动画片段 Root Motion 设置：**

三类设置属性相同，均位于 Animation Clip Import Settings 中：

1. **Root Transform Rotation**（Y 轴旋转）
2. **Root Transform Position (Y)**（垂直位移）
3. **Root Transform Position (XZ)**（水平位移）

**Bake Into Pose：**

- **勾选**：根骨骼变换不作为 Root Motion，而是作为普通动画烘焙到骨骼姿态中（蒙皮旋转，不带动 GameObject 旋转）
- **不勾选**：根骨骼变换作为 Root Motion，驱动 GameObject 的世界空间变换

> [!note] 典型场景
> 原地转身动画应勾选 Bake Into Pose（角色不实际旋转）；向前行走动画不应勾选（角色随动画位移）。

**Based Upon（方向参考）：**

决定动画开始时 GameObject 的朝向基准。推荐设置为 **Origin**（以模型导入时的朝向为准），配合 `OnAnimatorMove` 自定义处理。

**OnAnimatorMove 自定义控制：**

```csharp
void OnAnimatorMove() {
    // 手动应用 Root Motion 增量
    transform.position += animator.deltaPosition;
    transform.rotation *= animator.deltaRotation;
}
```

通过覆写 `OnAnimatorMove` 可以在代码层面完全控制 Root Motion 的插值与处理逻辑。

---

## Spine 骨骼动画

Spine 是 Esoteric Software 开发的 2D 骨骼动画工具，Unity 集成通过 `spine-unity` 运行时包实现。核心组件包括 **Skeleton**（骨骼数据）、**SkeletonAnimation**（动画播放器）、**AnimationState**（动画状态控制）和事件系统。

### Skeleton 组件

Skeleton 是 Spine 动画的数据载体，存储骨骼层级、槽位（Slot）、附件（Attachment）和皮肤（Skin）信息。

**设置 Attachment：**

```csharp
bool success = skeletonAnimation.Skeleton.SetAttachment("slotName", "attachmentName");

// 使用属性标签
[SpineSlot] public string slotProperty = "slotName";
[SpineAttachment] public string attachmentProperty = "attachmentName";
bool success = skeletonAnimation.Skeleton.SetAttachment(slotProperty, attachmentProperty);
```

**Setup Pose（重置姿态）：**

```csharp
skeleton.SetToSetupPose();        // 重置骨骼 + 槽位到初始姿态
skeleton.SetBonesToSetupPose();   // 仅重置骨骼
skeleton.SetSlotsToSetupPose();   // 仅重置槽位
```

**换肤（Skin）：**

```csharp
// 简单换肤
skeletonAnimation.Skeleton.SetSkin("skinName");
skeletonAnimation.Skeleton.SetSlotsToSetupPose();

// 组合皮肤（Mix-and-Match）
var skeleton = skeletonAnimation.Skeleton;
var skeletonData = skeleton.Data;
var mixAndMatchSkin = new Skin("custom-character");
mixAndMatchSkin.AddSkin(skeletonData.FindSkin("skin-base"));
mixAndMatchSkin.AddSkin(skeletonData.FindSkin("nose/short"));
mixAndMatchSkin.AddSkin(skeletonData.FindSkin("eyes/blue"));
mixAndMatchSkin.AddSkin(skeletonData.FindSkin("hair/black"));
skeleton.SetSkin(mixAndMatchSkin);
skeleton.SetSlotsToSetupPose();
skeletonAnimation.AnimationState.Apply(skeleton);
```

**运行时重打包（Repacking）：**

组合多皮肤时不可避免使用多种 Material，导致额外 Draw Call。使用 `Skin.GetRepackedSkin()` 将不同皮肤中的纹理区域合并为单张纹理图集，减少绘制调用：

```csharp
using Spine.Unity.AttachmentTools;

Skin repackedSkin = collectedSkin.GetRepackedSkin(
    "Repacked skin",
    skeletonAnimation.SkeletonDataAsset.atlasAssets[0].PrimaryMaterial,
    out Material runtimeMaterial,
    out SpineAtlasAsset runtimeAtlas
);
skeletonAnimation.Skeleton.Skin = repackedSkin;
skeletonAnimation.Skeleton.SetSlotsToSetupPose();
skeletonAnimation.AnimationState.Apply(skeleton);
```

**懒加载 Atlas 纹理：**

默认情况下所有 Atlas 纹理通过 SkeletonDataAsset 间接引用、随资源一起加载。使用 UPM 包可实现按需加载，减少初始内存占用。

### SkeletonAnimation 生命周期

SkeletonAnimation 将 Spine 骨骼挂载到 GameObject，负责播放动画、响应事件和更新渲染网格。

**脚本创建：**

```csharp
// 从 SkeletonDataAsset 创建
SkeletonAnimation instance = SkeletonAnimation.NewSkeletonAnimationGameObject(skeletonDataAsset);

// 运行时从原始数据创建（无需预先导入）
SpineAtlasAsset runtimeAtlas = SpineAtlasAsset.CreateRuntimeInstance(atlasTxt, textures, material, true);
SkeletonDataAsset runtimeSkeletonData = SkeletonDataAsset.CreateRuntimeInstance(skeletonJson, runtimeAtlas, true);
SkeletonAnimation instance = SkeletonAnimation.NewSkeletonAnimationGameObject(runtimeSkeletonData);
```

**更新生命周期回调：**

```
BeforeApply → UpdateLocal → UpdateComplete → UpdateWorld
```

| 回调 | 时机 | 用途 |
|:-----|:-----|:-----|
| **`BeforeApply`** | 动画应用到 Skeleton 之前 | 在动画应用前修改骨骼状态 |
| **`UpdateLocal`** | 动画更新完成、局部值写入骨骼后 | 读取或修改骨骼的局部值 |
| **`UpdateComplete`** | 所有骨骼世界值计算完成后 | 只读取骨骼世界值（不应修改） |
| **`UpdateWorld`** | 世界值计算后，触发 `skeleton.UpdateWorldTransform` 后再调用 | 根据世界值修改局部值（自定义约束场景） |

**SkeletonRenderer 回调：**

| 回调 | 时机 |
|:-----|:-----|
| **`OnRebuild`** | Skeleton 成功初始化后触发 |
| **`OnMeshAndMaterialsUpdated`** | LateUpdate 结束时，网格和所有 Material 更新完成后触发 |

**手动更新：**

- **`Update(deltaTime)`**：完整更新整个 Skeleton（推进 AnimationState、更新物理约束、应用动画、更新骨骼世界变换）。传入 deltaTime=0 可实现不推进时间的完整刷新
- **`ApplyAnimation()`**：仅重新应用动画到 Skeleton，不推进 AnimationState 时间、不更新物理约束

```csharp
// 重置姿态后不推进时间完成完整更新
skeleton.SetToSetupPose();
skeletonAnimation.Update(0);

// 更换槽位后仅重新应用动画
skeleton.SetSlotsToSetupPose();
skeletonAnimation.AnimationState.Apply(skeleton);
```

**脚本执行顺序控制：**

使用 `[DefaultExecutionOrder(-1)]` 可以在 SkeletonAnimation 之前执行自定义脚本：

```csharp
[DefaultExecutionOrder(-1)]
public class SetupPoseComponent : MonoBehaviour {
    void Update() {
        skeleton.SetToSetupPose();  // 每帧先重置姿态，再让 SkeletonAnimation 应用动画
    }
}
```

### AnimationState 控制

AnimationState 管理多轨道（Track）动画播放，支持播放、队列、混合和轨道条目定制。

**播放动画：**

```csharp
// 在轨道 0 循环播放 "walk" 动画
TrackEntry entry = skeletonAnimation.AnimationState.SetAnimation(0, "walk", true);

// 使用 AnimationReferenceAsset（推荐，避免字符串查找）
public AnimationReferenceAsset walkAnim;
TrackEntry entry = skeletonAnimation.AnimationState.SetAnimation(0, walkAnim, true);
```

**队列动画：**

```csharp
// 在轨道 0 当前动画之后，延迟 2 秒播放 "run"
TrackEntry entry = skeletonAnimation.AnimationState.AddAnimation(0, "run", true, 2f);
```

**空动画（Empty Animation）：**

空动画用于 mix-in（淡入）或 mix-out（淡出）到另一动画，清空轨道时保持骨骼最终姿态：

```csharp
// 设置空动画（mixDuration 秒内淡出）
TrackEntry entry = skeletonAnimation.AnimationState.SetEmptyAnimation(0, 0.3f);
// 队列空动画
entry = skeletonAnimation.AnimationState.AddEmptyAnimation(0, 0.3f, 1f);
// 清空指定轨道
skeletonAnimation.AnimationState.ClearTrack(0);
// 清空全部轨道
skeletonAnimation.AnimationState.ClearTracks();
```

**TrackEntry（轨道条目）：**

所有 AnimationState 方法返回 `TrackEntry` 对象，支持定制回放参数和订阅事件：

| 属性 | 说明 |
|:-----|:-----|
| **Animation** | 当前播放的动画引用 |
| **TrackIndex** | 所属轨道索引 |
| **TrackTime** | 轨道当前时间 |
| **TimeScale** | 时间缩放 |
| **MixDuration** | 混合持续时间 |
| **Alpha** | 轨道混合权重 |
| **EventThreshold** | 事件触发阈值 |

> [!warning] TrackEntry 生命周期
> 当动画从 AnimationState 中移除后，对应的 TrackEntry 失效并被 GC 回收。收到 Dispose 事件后不应再存储或访问该 TrackEntry。

### 事件系统

Spine 提供两层级事件订阅：**AnimationState 全局事件**（所有轨道）和 **TrackEntry 单实例事件**（指定播放实例）。

**事件类型：**

| 事件 | 说明 |
|:-----|:-----|
| **Start** | 动画开始播放 |
| **Interrupt** | 动画被中断（新动画替换或轨道被清空） |
| **Complete** | 动画无中断地完成一次播放（循环动画可多次触发） |
| **End** | 动画播放停止 |
| **Dispose** | 动画及其 TrackEntry 被销毁 |
| **Event** | Spine 编辑器中的用户自定义事件触发 |

**事件触发顺序：**

1. TrackEntry 事件**先**触发
2. 对应 AnimationState 事件**后**触发

**全局 vs 单实例：**

```csharp
// AnimationState 全局事件：监听所有轨道上的动画
skeletonAnimation.AnimationState.Start += OnAnyAnimationStart;
skeletonAnimation.AnimationState.Event += OnUserDefinedEvent;

// TrackEntry 单实例事件：仅监听该次播放
TrackEntry entry = skeletonAnimation.AnimationState.SetAnimation(0, "attack", false);
entry.Complete += OnAttackComplete;
entry.Event += OnAttackEvent;
```

**事件驱动的音频（性能优化）：**

```csharp
// 缓存 EventData 引用，避免每次字符串比较
Spine.EventData footstepEventData;

void Start() {
    footstepEventData = skeletonAnimation.Skeleton.Data.FindEvent("footstep");
    skeletonAnimation.AnimationState.Event += HandleEvent;
}

void HandleEvent(TrackEntry trackEntry, Spine.Event e) {
    if (e.Data == footstepEventData) {  // 引用比较，零 GC 开销
        PlayFootstepSound();
    }
}
```

### Material 与换装

Spine 角色换装涉及 Material、Texture 和 Attachment 的管理，需权衡性能与灵活性。

**多种 CanvasRenderer 模式：**

开启 Advanced → Multiple CanvasRenderers 后，每个 Material 对应一个独立子 GameObject（CanvasRenderer），用于分离渲染批次。

**Material 替换方法：**

| 方法 | 机制 | 适用场景 |
|:-----|:-----|:-----|
| **Skin 换皮** | 切换预定义的 Skin | 最简单，需美术支持 |
| **手动 Attachment** | 代码切换槽位 Attachment | 万能，但性能开销大 |
| **CustomTextureOverride** | 替换指定槽位的纹理 | 需原始图片尺寸一致 |
| **MaterialPropertyBlocks** | 修改 Shader 属性值 | 性能最优，但不支持 SkeletonGraphic |

**CustomMaterialOverride / CustomTextureOverride：**

```csharp
// Material 替换
skeletonAnimation.CustomMaterialOverride[originalMaterial] = newMaterial;
skeletonAnimation.CustomMaterialOverride.Remove(originalMaterial);

// SkeletonGraphic 的 Texture 替换
skeletonGraphic.CustomTextureOverride[originalMaterial] = newTexture;
```

**MaterialPropertyBlocks：**

```csharp
MaterialPropertyBlock mpb = new MaterialPropertyBlock();
mpb.SetColor("_FillColor", Color.red);
mpb.SetFloat("_FillPhase", 1.0f);
GetComponent<MeshRenderer>().SetPropertyBlock(mpb);

// 清除覆盖
mpb.Clear();
GetComponent<Renderer>().SetPropertyBlock(mpb);
```

> [!warning] 合批限制
> 使用不同的 MaterialPropertyBlock 参数值会**破坏合批（Batching）**。仅当所有对象的 PropertyBlock 参数完全相同时才可合批渲染。

**性能建议：**

- 缓存 `Shader.PropertyToID("属性名")` 的 int 结果，避免每次通过字符串访问属性
- 缓存 MaterialPropertyBlock 实例，复用而非每次新建
- 预分拆各部位到不同 Atlas（自动生成对应 Material），方便后期独立更换

**换装策略对比：**

| 策略 | 复杂度 | 性能 | 灵活性 |
|:-----|:-----|:-----|:-----|
| **Skin** | 低 | 高 | 低（需美术预定义） |
| **手动 Attachment** | 中 | 低 | 高 |
| **CustomTextureOverride** | 中 | 中 | 中（需同尺寸纹理） |
| **MaterialPropertyBlocks** | 高 | 最高 | 中（不支持 Graphic） |
