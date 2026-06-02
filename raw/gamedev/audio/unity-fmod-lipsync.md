---
title: "Unity FMOD 音频与 LipSync 唇形同步"
type: source
updated: 2026-06-02
tags:
  - unity
  - audio
  - fmod
  - lipsync
  - dsp
---

# Unity FMOD 音频与 LipSync 唇形同步

## FMOD 音频系统

FMOD 是 Unity 中广泛使用的专业音频中间件，提供 Event（事件）、Bus（总线）、Parameter（参数）、VCA（音量控制）、Snapshot（快照）等完整的音频管理能力，支持 DSP 效果处理和动态混音。

### Event 事件

FMOD 中所有音频效果都依赖于 Event。Event 是 FMOD Studio 中设计好的音频资源的基本播放单元。

从程序角度看，Event 分为两种类型：

| 类型 | 说明 |
|:-----|:-----|
| 功能性事件 | 控制某一类或某个音频的暂停、恢复等播放状态 |
| 音频类事件 | 常规的声音播放 |

**EventInstance 生命周期：**

`CreateInstance` → `start` → `release` → 异步销毁。

```csharp
private EventInstance eventInstance;
private string eventPath = "event:/MySoundEvent";

void Start()
{
    eventInstance = RuntimeManager.CreateInstance(eventPath);
    eventInstance.start();
    eventInstance.release();
}
```

- `release()` 标记事件实例为可释放状态。实例一旦进入 `FMOD_STUDIO_PLAYBACK_STOPPED` 状态，异步更新系统自动销毁。
- 一般 `start` 之后立即调用 `release` 进行标记释放。
- 如需多次触发同一事件，可使用 `step` 停止后重新 `start`，或创建新的 EventInstance。
- 调用 `release` 后仍可继续操作，但如果已完全停止，可能返回无效句柄。

### Bus 总线体系

Bus 是相关音频或功能事件的集合总控，可以统一控制一组事件的各类参数。FMOD 提供了三层 Bus 架构：Master Bus、Group Bus、Return Bus。

#### Master Bus（主总线）

Master Bus 是 FMOD 音频信号流的最终出口，所有音频最终经过 Master Bus 混音后输出到音频设备。

- 控制整个游戏的主音量，用于全局音频管理
- **不可跳过**，FMOD 需要它完成最终混音和输出
- 适用场景：统一控制全局音量、处理最终混音（动态范围控制、压缩）、全局静音/暂停

```csharp
Bus musicBus = RuntimeManager.GetBus("bus:/Music");
musicBus.setVolume(volume);    // 0.0 ~ 1.0
musicBus.setMute(mute);
```

#### Group Bus（分组总线）

Group Bus 用于分类管理不同类型的音效，允许单独调整某类音效的音量、混响、EQ。

常见分组：

| Bus | 用途 |
|:----|:-----|
| Music Bus | 背景音乐 |
| SFX Bus | 音效（UI、环境、战斗） |
| Voice Bus | 角色语音/对话 |

**信号流向：**

```text
[Event: Background Music] → [Music Bus] → [Master Bus] → [音频输出]
[Event: Footstep SFX]    → [SFX Bus]   → [Master Bus] → [音频输出]
[Event: Character Voice] → [Voice Bus] → [Master Bus] → [音频输出]
```

#### Return Bus（返回总线）

Return Bus 用于处理音频特效，接收来自多个 Group Bus 的 Send 信号，统一处理后将结果返回 Master Bus。

- **音效不直接流入 Return Bus**，而是由 Group Bus 通过 Send 发送部分信号
- 多个 Group Bus 可共享同一个 Return Bus，节省 CPU 计算量
- 适用场景：洞穴/教堂混响、水下低通滤波、全局特效处理

```text
[Event: 角色脚步声] → [SFX Bus] → [Reverb Return Bus] → [Master Bus] → [音频输出]
[Event: 枪声]      → [SFX Bus] ↗
[Event: 环境音]    → [SFX Bus] ↗
```

**三类 Bus 对比：**

| Bus 类型 | 作用 | 信号流向 | 适用场景 |
|:---------|:-----|:---------|:---------|
| Master Bus | 最终音频输出与混音 | Group Bus → Master Bus → 音频输出 | 全局音量、全局混音 |
| Group Bus | 分类管理音效 | Event → Group Bus → Master Bus | 音乐/SFX/语音分组 |
| Return Bus | 共享 DSP 特效 | Group Bus → Return Bus → Master Bus | 混响、滤波、全局特效 |

> [!warning] Bus 串联
> Bus 是串联连接，如需链路畅通，路径上所有 Bus 必须保持闭合状态。

### 参数系统

FMOD 参数允许在运行时动态调整音频行为，分为事件参数（Event Parameter）和全局参数（Global Parameter）。

| 参数类型 | 作用范围 | 适用场景 | 调用方式 |
|:---------|:---------|:---------|:---------|
| 事件参数 | 特定事件实例 | 单个音效的动态变化（如脚步音随地形变化） | `eventInstance.setParameterByName("参数名", 值)` |
| 全局参数 | 所有事件，多事件共享 | 影响整个游戏环境（天气、战斗状态、情绪） | `RuntimeManager.StudioSystem.setParameterByName("参数名", 值)` |

**使用示例：**

```csharp
public class FMODParameterExample : MonoBehaviour
{
    public void SetFootstepSurface(float surfaceType)
    {
        EventInstance footstepInstance = RuntimeManager.CreateInstance("event:/Footstep");
        footstepInstance.setParameterByName("SurfaceType", surfaceType);
        footstepInstance.start();
        footstepInstance.release();
    }

    public void SetWeatherCondition(string weather)
    {
        EventInstance ambienceInstance = RuntimeManager.CreateInstance("event:/Ambience");
        ambienceInstance.setParameterByNameWithLabel("Weather", weather);
        ambienceInstance.start();
        ambienceInstance.release();
    }
}
```

- 事件参数：改变同一音效的表现（如脚步声在草地、木板、金属上的不同声音）
- 全局参数：改变整个游戏环境（如雨天音效增强、战斗音乐情绪变化）

### VCA 音量控制

VCA（Voltage Controlled Amplifier）是 FMOD 中的音量控制单元，用于集中调节一组 Bus 的音量，类似于一组 Bus 的"总音量推子"。

**VCA 特点：**

| 特性 | 描述 |
|:-----|:-----|
| 控制对象 | 多个 Bus 的最终输出音量 |
| DSP 支持 | ❌ 不支持混响、滤波等特效 |
| 层级结构 | ❌ 无嵌套，所有 VCA 平级 |
| 场景管理 | 适合 UI 设置中的"音乐/语音/音效"滑块 |
| 性能开销 | 极小，仅做音量乘法，不参与信号处理 |

```csharp
public class FMODVCAController : MonoBehaviour
{
    public void SetVCAVolume(float volume)
    {
        VCA vca = RuntimeManager.GetVCA("vca:/MusicVCA");
        vca.setVolume(volume);  // 0.0 ~ 1.0
    }
}
```

**Bus vs VCA 对比：**

| 对比点 | Bus | VCA |
|:-------|:----|:----|
| 控制内容 | 音频信号流、混音、DSP | 仅音量控制 |
| DSP 支持 | ✅ 可添加混响、滤波等 | ❌ 不支持 |
| 层级结构 | ✅ 树状嵌套 | ❌ 平级无嵌套 |
| 跨层控制 | ❌ 仅控制自身及子 Bus | ✅ 可跨任意 Bus 层级 |
| 用途 | 路由、混响、暂停、效果管理 | 统一逻辑音量控制 |

**VCA 的跨层控制能力：**

FMOD 的 Bus 是树状结构，设置 `bus:/SFX` 仅影响其自身与子 Bus。VCA 可将多个不相关层级的 Bus 绑定在一起：

```text
Master Bus
├── Music
├── SFX
│   ├── UI
│   └── Gameplay
└── UI
    └── Popup
```

`vca:/UIVolume` 可同时控制 `bus:/SFX/UI` 和 `bus:/UI/Popup`，即便它们不在同一 Bus 分支下。

> [!tip] VCA 是游戏中 UI 设置（音效、语音、背景音乐）音量滑块的首选方案。

### 事件回调

当事件实例被销毁时，FMOD 触发 `EVENT_CALLBACK_TYPE.DESTROYED` 回调，适用于销毁前清理、防止内存泄漏、日志记录等场景。

```csharp
using FMOD;
using FMOD.Studio;
using FMODUnity;
using UnityEngine;
using System;
using System.Runtime.InteropServices;

public class FMODEventDestroyCallback : MonoBehaviour
{
    private EventInstance eventInstance;

    [AOT.MonoPInvokeCallback(typeof(EVENT_CALLBACK))]
    private static RESULT EventCallback(EVENT_CALLBACK_TYPE type, IntPtr instancePtr, IntPtr parameters)
    {
        EventInstance eventInstance = new EventInstance(instancePtr);

        if (type == EVENT_CALLBACK_TYPE.DESTROYED)
        {
            eventInstance.getDescription(out EventDescription eventDesc);
            eventDesc.getPath(out string eventPath);
            // 执行清理逻辑
        }

        return RESULT.OK;
    }

    void Start()
    {
        eventInstance = RuntimeManager.CreateInstance("event:/MyEvent");
        eventInstance.setCallback(EventCallback, EVENT_CALLBACK_TYPE.DESTROYED);
        eventInstance.start();
        eventInstance.release();
    }
}
```

> [!warning] 回调线程限制
> 事件回调在音频线程执行，**不能直接调用 Unity API**（如 `GameObject`、`Transform` 等）。如需与主线程交互，需通过线程安全队列中转。

### Master Bank 与 Strings Bank

**Master Bank（主 Bank）** 是 FMOD 项目的核心 Bank，包含全局混音器（Global Mixer），管理整个项目的音频路由和音效输出。

| 组件 | 作用 |
|:-----|:-----|
| Mixer（混音器） | 管理 Bus，控制全局音量和效果 |
| Snapshots（快照） | 存储不同混音状态，如战斗/探索模式 |
| Global Parameters（全局参数） | 控制整个游戏的音效环境 |

- **所有 EventInstance 依赖 Master Bank**，即使事件的 `.bank` 已加载，Master Bank 未加载则事件仍无法播放
- 建议在游戏启动时加载 Master Bank

**Strings Bank（字符串 Bank）** 存储事件、Bus、参数路径到 GUID 的映射。FMOD 内部使用 GUID 访问事件，Strings Bank 负责运行时查找。

| 功能 | 描述 |
|:-----|:-----|
| 事件路径 → GUID 映射 | 允许用字符串访问事件，无需手动维护 GUID |
| Bus & Parameter 路径解析 | 让 `RuntimeManager.GetBus("bus:/Music")` 正常工作 |
| 内存占用 | 仅存储映射信息，不含音频数据，不影响内存 |

```csharp
// 通过字符串获取 GUID
FMOD.GUID eventGUID;
RuntimeManager.StudioSystem.lookupID(eventPath, out eventGUID);

// 使用 GUID 创建事件
EventInstance eventInstance;
RuntimeManager.StudioSystem.createEventInstance(ref eventGUID, out eventInstance);
```

> [!note] Master Bank 负责全局混音，Strings Bank 让开发者用字符串访问事件。两者通常需一起加载。

### 快照（Snapshot）

快照是 FMOD 提供的动态混音调整机制，允许在不同游戏状态下同时修改多个 Bus 的音量、效果参数。它的作用类似一组**预设（Preset）**，可在运行时激活/停用。

**核心机制：** 快照类似音频场景的切换：

1. 进入战斗模式 → 提高战斗音效，降低环境音
2. 进入洞穴 → 增加混响，声音更有空间感
3. 角色入水 → 启用低通滤波，声音变沉闷

**快照的特点：**

- 不创建新 Bus，仅调整已有 Bus 的参数
- 多个快照可同时生效，也可互相过渡

```csharp
public class FMODSnapshotController : MonoBehaviour
{
    private EventInstance snapshot;

    public void ActivateSnapshot()
    {
        snapshot = RuntimeManager.CreateInstance("snapshot:/BattleMode");
        snapshot.start();
    }

    public void DeactivateSnapshot()
    {
        snapshot.stop(FMOD.Studio.STOP_MODE.ALLOWFADEOUT);
    }
}
```

**适用场景：**

| 场景 | 作用 |
|:-----|:-----|
| 战斗模式切换 | 提高战斗音效音量，降低环境音，增强紧张感 |
| 室内/室外过渡 | 进入洞穴/房间时调整音量与混响 |
| 天气变化 | 下雨时增加风声，降低背景音乐 |
| 水下效果 | 应用低通滤波，声音更闷 |
| 暂停游戏 | 降低所有音效音量 |
| 角色受伤状态 | 降低环境音，提高心跳/呼吸声 |

### 延音点（Sustain Point）

延音点是 FMOD 时间轴中的控制标记。播放到延音点时事件暂停（或循环），直到程序调用 `keyOff()` 后才继续播放后续内容。

**延音点 + Loop 区联合机制：**

```text
[Intro] → [Loop] 🔁 → 🔷 延音点 → [Release 结尾段]
```

调用 `keyOff()` 后，播放跳出循环，从延音点继续播放 Release 段。

**播放逻辑：**

| 状态 | 行为 |
|:-----|:-----|
| 播放到延音点前 | 音频正常播放 |
| 播放到延音点 | 暂停或循环（取决于是否设置 loop） |
| 调用 `keyOff()` 后 | 继续播放延音点之后的内容 |
| 延音点后无内容 | `keyOff()` 后播放可能立即结束 |

```csharp
public class FMODSustainController : MonoBehaviour
{
    private EventInstance sustainEvent;

    public void StartChargingSound()
    {
        sustainEvent = RuntimeManager.CreateInstance("event:/Skill/Charge");
        sustainEvent.start();
        sustainEvent.release();
    }

    public void ReleaseChargeSound()
    {
        sustainEvent.keyOff(); // 触发延音点之后播放
    }
}
```

**适用与局限：**

| 控制目标 | 是否推荐 |
|:---------|:---------|
| 精准释放动作同步 | ❌ 不推荐——无法立即跳出，只能等播放头走到 sustain 点 |
| 渐进/节奏性过渡音效 | ✅ 推荐（蓄力技能、音乐结构控制） |
| 音乐结构控制 | ✅ 推荐 |
| 剧情控制演出音效 | ✅ 推荐 |

### UserData

`UserData` 允许在播放事件时附带自定义数据（字符串、对象引用），并在 FMOD 回调中取回。常用于回调上下文、事件标记、性能分析。

**实现原理：** 通过 `GCHandle.Alloc()` 将 C# 托管对象固定，防止 GC 回收或移动内存地址，再通过 `GCHandle.ToIntPtr()` 转为原生指针绑定到 FMOD 事件。

```csharp
// 设置 UserData
string userData = "Hello User Data!";
GCHandle handle = GCHandle.Alloc(userData);
IntPtr pointer = GCHandle.ToIntPtr(handle);
sound.setUserData(pointer);

// 获取并释放 UserData
IntPtr pointer;
sound.getUserData(out pointer);
GCHandle handle = GCHandle.FromIntPtr(pointer);
string userData = handle.Target as string;
handle.Free();  // 必须释放，防止内存泄漏
```

> [!warning] 注意事项
> - 每个 `EventInstance` 只能绑定一个 `UserData`
> - 必须调用 `handle.Free()`，否则内存泄漏
> - struct 会装箱成 object，可能导致访问麻烦，不推荐传值类型

---

## FMOD DSP 数字信号处理

FMOD 提供 DSP（Digital Signal Processor）系统，允许开发者挂载数字信号处理器到音频通道上，实现实时频谱分析、音效修改、数据监控等功能。

### ChannelGroup 通道组

ChannelGroup 可附着在音频 EventInstance 上，用于添加 DSP 进行声音监控或修改。通过 `eventInstance.getChannelGroup()` 获取，随后可在其上挂载和移除 DSP。

### DSP 创建与配置

DSP 的完整生命周期：创建 → 配置参数 → `flushCommands()` → 附加到 ChannelGroup → 使用 → 移除 → 释放。

```csharp
private DSP _dsp;
private DSP_PARAMETER_FFT _fftParam = default;
private ChannelGroup _channelGroup = default;

private DSP CreateDSP(EventInstance eventInstance)
{
    DSP dsp;
    FMOD.RESULT result;

    // 1. 创建指定类型的 DSP（此处为 FFT 频谱分析）
    result = FMODUnity.RuntimeManager.CoreSystem.createDSPByType(FMOD.DSP_TYPE.FFT, out dsp);
    if (result != FMOD.RESULT.OK)
        ReportError("Create DSP failed: ");

    // 2. 配置 DSP 参数：窗函数类型
    result = dsp.setParameterInt((int)FMOD.DSP_FFT.WINDOW, (int)WINDOW_TYPE);
    if (result != FMOD.RESULT.OK)
        ReportError("Set DSP parameter failed: ");

    // 3. 配置 DSP 参数：窗口大小（2048、4096 等，乘 2 表示双倍精度）
    result = dsp.setParameterInt((int)FMOD.DSP_FFT.WINDOWSIZE, WINDOW_SIZE * 2);
    if (result != FMOD.RESULT.OK)
        ReportError("Set DSP parameter failed: ");

    // 4. 刷新命令队列，确保参数生效
    FMODUnity.RuntimeManager.StudioSystem.flushCommands();

    // 5. 获取事件的 ChannelGroup 并附加 DSP
    result = eventInstance.getChannelGroup(out ChannelGroup channelGroup);
    if (result != FMOD.RESULT.OK)
        ReportError("Get ChannelGroup from EventInstance failed: ");
    _channelGroup = channelGroup;

    result = channelGroup.addDSP(FMOD.CHANNELCONTROL_DSP_INDEX.HEAD, dsp);
    if (result != FMOD.RESULT.OK)
        ReportError("Add DSP to ChannelGroup failed: ");

    return dsp;
}

private void ClearDSP()
{
    FMOD.RESULT result;
    _channelGroup.removeDSP(_dsp);   // 先从 ChannelGroup 上移除
    result = _dsp.release();          // 再释放 DSP 资源
    if (result != FMOD.RESULT.OK)
        ReportError("Release DSP failed: ");
    _dsp = default;
    _channelGroup = default;
}
```

**关键注意事项：**

1. `setParameterXXX` 调用后必须执行 `flushCommands()` 使参数生效
2. 清理时**必须先移除再释放**：`channelGroup.removeDSP` → `dsp.release`
3. DSP 的 `DSP_TYPE.FFT` 类型用于快速傅里叶变换频谱分析

### 频谱分析（FFT）

通过 FFT 类型的 DSP，可获取音频的实时频谱数据，用于可视化、音频驱动效果、唇形同步等场景。

```csharp
private void GetSpectrumData()
{
    if (!_dsp.hasHandle()) return;

    System.IntPtr _data;
    uint _length;

    // 获取 DSP 参数数据（频谱）
    if (_dsp.getParameterData((int)FMOD.DSP_FFT.SPECTRUMDATA, out _data, out _length) != FMOD.RESULT.OK)
        return;

    // 从非托管内存还原为 DSP_PARAMETER_FFT 结构体
    _fftParam = (FMOD.DSP_PARAMETER_FFT)Marshal.PtrToStructure(_data, typeof(DSP_PARAMETER_FFT));

    if (_fftParam.numchannels > 0)
    {
        // 获取第一个通道的频谱数据
        _fftParam.getSpectrum(0, ref _spectrum);
    }
}
```

**获取频谱的关键步骤：**

1. 调用 `dsp.getParameterData(DSP_FFT.SPECTRUMDATA, out data, out length)` 获取数据指针
2. 通过 `Marshal.PtrToStructure` 将非托管内存转换为 `DSP_PARAMETER_FFT`
3. 检查 `numchannels > 0` 确认有有效数据
4. 调用 `getSpectrum(channelIndex, ref spectrumArray)` 提取具体通道频谱

> [!note] DSP 系统是构建音频驱动功能（频谱可视化、节拍检测、唇形同步）的核心基础设施。频谱数据经由 FFT 变换后，可作为后续音频分析算法的输入。

---

## LipSync 唇形同步

唇形同步（LipSync）技术根据音频输入实时检测当前发音的元音，驱动角色的口型动画。本方案采用**共振峰估计（Formant Estimation）** 结合**高斯相似度分类**，并利用 Unity JobSystem 实现高性能并行处理。

### 共振峰方案概述

该方案完整实现以下技术栈：

| 技术层 | 方法 | 目的 |
|:-------|:-----|:-----|
| 频谱预处理 | 高斯滤波卷积 + 帧内归一化 | 平滑频谱，减小能量波动影响 |
| 能量过滤 | EMA 自适应静音阈值 | 丢弃低能量帧，减少误识别 |
| 帧间平滑 | 指数平滑（能量/置信度） | 降低抖动，避免口型突变 |
| 核心分类 | 共振峰估计 (F1/F2) + 高斯相似度 | 区分 A/E/I/O/U 五个元音 |
| 回退机制 | 能量频带法 | 低置信度时回退到能量分布判定 |
| 并行加速 | Unity JobSystem + Burst | 卷积、归一化、能量计算、分类并行 |

**文件结构（基于 `com.hero.lipsync` 包）：**

| 文件 | 职责 |
|:-----|:-----|
| `LipSync_Algo.cs` | 核心算法：高斯滤波、卷积、峰值查找、频带能量、共振峰估计与分类 |
| `LipSync_Job.cs` | Job 定义：IJobParallelFor 卷积、IJob 归一化、IJobParallelFor 能量法、IJob 共振峰分类 |
| `LipSync_JobEnv.cs` | 调度编排：内存分配/释放、Job 链调度、EMA 自适应、帧间平滑、结果输出 |

### 信号处理管线

完整的每帧处理流程如下：

```text
原始 Spectrum
    │
    ▼
[高斯卷积平滑] ─── IJobParallelFor ─── 每个频点独立卷积
    │
    ▼
[帧内归一化] ─── IJob ─── NormalizeSpectrum：除以最大值，消除能量波动
    │
    ▼
[语音清晰度评分] ─── IJob ─── GetClarity：300-3400Hz 频段能量占比
    │                                    ↓
    ▼                        ┌──────────────────────┐
[并 行 分 支]               │ 能量法（IJobParallelFor）│
    │                       │ 每个元音独立计算频带能量   │
    ├── [共振峰分类]         │ 频率加权（元音频段 1.5x） │
    │    IJob               │ 乘 clarity 评分          │
    │    EstimateFormants   └──────────────────────┘
    │    ClassifyVowelByFormants
    │         │
    ▼         ▼
[置信度判断]
    │
    ├── confidence > 0.22 → 采用共振峰结果
    └── confidence ≤ 0.22 → 回退能量法 + 帧间指数平滑
    │
    ▼
[输出元音索引] ─── 驱动 BlendShape / 骨骼动画
```

**关键技术细节：**

1. **帧内归一化（NormalizeSpectrum）**：将平滑后的频谱除以该帧最大值，使频谱形状而非绝对能量决定判定结果，减少音量波动对识别的影响

2. **EMA 自适应静音阈值**：每帧计算频谱能量和，使用指数移动平均（`α = 0.2`）追踪能量趋势，自适应阈值为 `max(baseThreshold, EMA × 0.35)`，丢弃低于阈值的帧

3. **帧间指数平滑**：保存上一帧的 peak 值和置信度，用 `Mathf.Lerp(prev, current, alpha)` 做帧间平滑，避免相邻帧口型剧烈切换

### 共振峰估计与分类

#### EstimateFormants — 共振峰估计

从频谱中寻找前两个局部峰值作为第一共振峰（F1）和第二共振峰（F2），返回频率值（Hz）。

```csharp
[BurstCompile]
public static void EstimateFormants(NativeArray<float> spectrum, int sampleRate,
    out float f1, out float f2)
{
    f1 = 0f; f2 = 0f;
    int n = spectrum.Length;
    if (n < 3) return;

    float top1Val = 0f, top2Val = 0f;
    int top1Idx = -1, top2Idx = -1;

    // 遍历频谱找局部峰值（当前值 > 左右邻值）
    for (int i = 1; i < n - 1; i++)
    {
        if (spectrum[i] > spectrum[i - 1] && spectrum[i] > spectrum[i + 1])
        {
            float v = spectrum[i] * spectrum[i];  // 平方增强峰值差异
            if (v > top1Val)
            {
                top2Val = top1Val; top2Idx = top1Idx;
                top1Val = v; top1Idx = i;
            }
            else if (v > top2Val)
            {
                top2Val = v; top2Idx = i;
            }
        }
    }

    // 索引转频率：freq = index × nyquist / spectrumLength
    float nyquist = sampleRate / 2f;
    if (top1Idx >= 0)
    {
        float freq1 = top1Idx * nyquist / n;
        if (top2Idx >= 0)
        {
            float freq2 = top2Idx * nyquist / n;
            // 确保 F1 ≤ F2
            if (freq1 <= freq2) { f1 = freq1; f2 = freq2; }
            else { f1 = freq2; f2 = freq1; }
        }
        else { f1 = freq1; f2 = 0f; }
    }
}
```

**算法要点：**
- 局部峰值判定：`spectrum[i] > spectrum[i-1] && spectrum[i] > spectrum[i+1]`
- 平方加权：增强高能量峰的区分度
- 取前两个最大峰值作为 F1/F2
- 索引 → 频率转换：`freq = index × (sampleRate / 2) / spectrumLength`

#### ClassifyVowelByFormants — 高斯相似度分类

基于 F1/F2 估计值，计算与五个标准元音参考均值的二维高斯相似度，返回最匹配的元音索引和置信度。

**元音参考均值（F1, F2 单位 Hz）：**

| 元音 | F1 | F2 | 特征说明 |
|:-----|:---|:---|:---------|
| A | 800 | 1200 | 开口最大，F1 高 |
| E | 400 | 2000 | F2 较高，F1 中等偏低 |
| I | 300 | 2400 | F1 最低，F2 最高 |
| O | 500 | 1000 | F1/F2 均中等 |
| U | 350 | 800 | F1/F2 均较低 |

```csharp
public static int ClassifyVowelByFormants(float f1, float f2, out float confidence)
{
    confidence = 0f;
    if (f1 <= 0f || f2 <= 0f) return -1;

    // 五个元音的标准 (F1, F2) 参考均值
    float[,] means = new float[5, 2]
    {
        { 800f, 1200f }, // A
        { 400f, 2000f }, // E
        { 300f, 2400f }, // I
        { 500f, 1000f }, // O
        { 350f, 800f }   // U
    };

    float f1Sigma = 200f;   // F1 标准差
    float f2Sigma = 400f;   // F2 标准差（元音 F2 变化范围更大）

    float[] scores = new float[5];
    float sum = 0f;
    for (int i = 0; i < 5; i++)
    {
        float d1 = (f1 - means[i, 0]) / f1Sigma;
        float d2 = (f2 - means[i, 1]) / f2Sigma;
        float s = Mathf.Exp(-0.5f * (d1 * d1 + d2 * d2));  // 二维高斯核
        scores[i] = s;
        sum += s;
    }

    if (sum <= 0f) return -1;

    // 取归一化后最高分
    int best = -1;
    float bestScore = 0f;
    for (int i = 0; i < 5; i++)
    {
        float normalized = scores[i] / sum;
        if (normalized > bestScore)
        {
            bestScore = normalized;
            best = i;
        }
    }

    confidence = bestScore;
    return best;  // 返回元音索引（0=A, 1=E, 2=I, 3=O, 4=U）
}
```

**分类流程：**
1. 计算观测值 (F1, F2) 到每个元音参考均值的马氏距离
2. 应用二维高斯核：`exp(-0.5 × (d1² + d2²))`，其中 `d1 = (F1 - μ_F1) / σ_F1`，`d2 = (F2 - μ_F2) / σ_F2`
3. Softmax 归一化：各得分除以总和
4. 输出最高分对应的元音索引及归一化得分作为置信度

### JobSystem 并行化

为满足实时唇形同步的性能要求，整个管线使用 Unity JobSystem 并行化处理。

#### Job 定义

**卷积 Job（IJobParallelFor）** — 每个频点独立执行高斯模糊卷积：

```csharp
[BurstCompile]
public struct GetSmoothSpectrum : IJobParallelFor
{
    [ReadOnly] public int filterLength;
    [ReadOnly] public EPaddleType paddleType;
    [ReadOnly] public VowelInfomation vowelInfomation;
    [NativeDisableParallelForRestriction]
    [ReadOnly] public NativeArray<float> Spectrum;
    [NativeDisableParallelForRestriction]
    public NativeArray<float> SmoothedSpectrum;

    public void Execute(int index)
    {
        LipSync_Algo.Convolute(Spectrum, vowelInfomation.GaussianFilter,
            paddleType, SmoothedSpectrum, filterLength, index);
    }
}
```

**归一化 Job（IJob）** — 全频谱归一化（查找最大值后除以最大值）：

```csharp
[BurstCompile]
public struct NormalizeSpectrum : IJob
{
    public NativeArray<float> SmoothedSpectrum;

    public void Execute()
    {
        float maxVal = 0f;
        for (int i = 0; i < SmoothedSpectrum.Length; i++)
            if (SmoothedSpectrum[i] > maxVal) maxVal = SmoothedSpectrum[i];

        if (maxVal <= 1e-6f) return;
        float inv = 1f / maxVal;
        for (int i = 0; i < SmoothedSpectrum.Length; i++)
            SmoothedSpectrum[i] *= inv;
    }
}
```

**清晰度 Job（IJob）** — 计算 300-3400Hz 语音频段能量占比：

```csharp
[BurstCompile]
public struct GetClarity : IJob
{
    [ReadOnly] public int sampleRate;
    [ReadOnly] public NativeArray<float> smoothedSpectrum;
    public NativeArray<float> speechClarity;

    public void Execute()
    {
        speechClarity[0] = LipSync_Algo.CalculateFrequencyBandRatio(
            smoothedSpectrum, 300f, 3400f, sampleRate);
    }
}
```

**能量法 Job（IJobParallelFor）** — 每个元音并行计算加权频带能量：

```csharp
[BurstCompile]
public struct LipSyncJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float> speechClarity;
    [ReadOnly] public int sampleRate;
    [ReadOnly] public VowelInfomation vowelInfomation;
    [ReadOnly] public NativeArray<float> SmoothedSpectrum;
    public NativeArray<float> peakValue;

    public void Execute(int index)
    {
        peakValue[index] = LipSync_Algo.GetFrequencyBandEnergy(SmoothedSpectrum,
            vowelInfomation.POfBegin[index], vowelInfomation.POfEnd[index],
            sampleRate) * speechClarity[0];
        peakValue[index] = math.pow(peakValue[index], vowelInfomation.PowOfAEIOU);
    }
}
```

**共振峰分类 Job（IJob）** — 单帧共振峰估计与分类：

```csharp
[BurstCompile]
public struct FormantClassifyJob : IJob
{
    [ReadOnly] public NativeArray<float> SmoothedSpectrum;
    [ReadOnly] public int sampleRate;
    public NativeArray<int> DetectedVowel;
    public NativeArray<float> DetectedConfidence;

    public void Execute()
    {
        float f1, f2;
        LipSync_Algo.EstimateFormants(SmoothedSpectrum, sampleRate, out f1, out f2);
        float conf;
        int idx = LipSync_Algo.ClassifyVowelByFormants(f1, f2, out conf);
        DetectedVowel[0] = idx;
        DetectedConfidence[0] = conf;
    }
}
```

#### 调度编排（JobEnv）

`LipSync_JobEnv` 负责整合所有 Job 的调度顺序、内存管理和结果输出：

```text
调度顺序（Job 链）：
                    ┌──────────────────┐
                    │ GetSmoothSpectrum │  (IJobParallelFor, batch=64)
                    └────────┬─────────┘
                             │ getSmoothHandle
                             ▼
                    ┌──────────────────┐
                    │ NormalizeSpectrum │  (IJob)
                    └────────┬─────────┘
                             │ normalizeHandle
                             ▼
                    ┌──────────────────┐
                    │   GetClarity     │  (IJob)
                    └────────┬─────────┘
                             │ getClarityHandle
                  ┌──────────┴──────────┐
                  ▼                     ▼
    ┌──────────────────────┐  ┌──────────────────────┐
    │    LipSyncJob        │  │  FormantClassifyJob  │
    │  (IJobParallelFor)   │  │      (IJob)          │
    └──────────┬───────────┘  └──────────┬───────────┘
               │                         │
               └──────────┬──────────────┘
                          │ JobHandle.CombineDependencies
                          ▼
                  ┌──────────────────┐
                  │   主线程等待       │
                  │ Complete()        │
                  └────────┬─────────┘
                           ▼
                  ┌──────────────────┐
                  │  置信度判断输出    │
                  │  Formant vs 能量法 │
                  └──────────────────┘
```

**关键设计决策：**

1. **并发执行**：能量法（LipSyncJob）和共振峰分类（FormantClassifyJob）在 `GetClarity` 完成后并行执行，通过 `JobHandle.CombineDependencies` 合并等待
2. **EMA 自适应阈值**：每帧维护指数移动平均值，阈值随音频能量动态调整，避免固定阈值在低/高音量场景失效
3. **双向帧间平滑**：能量法的 peak 值（`Lerp(prev, cur, 0.6)`）和共振峰置信度（`Lerp(prev, cur, 0.5)`）均做帧间平滑
4. **NativeArray 生命周期**：每帧 `Allocate(TempJob)` → Schedule → Complete → `Dispose`，避免内存泄漏

#### 元音频率权重

`GetFrequencyBandEnergy` 在计算各元音频带能量时，对特定频率范围施加 1.5× 权重：

| 频段 | 范围 | 关系 |
|:-----|:-----|:-----|
| 低频元音区域 | 300–1100 Hz | 覆盖 A、O、U 的 F1 区域 |
| 高频元音区域 | 2000–3000 Hz | 覆盖 E、I 的 F2 区域 |

权重增强使元音特征频段的能量更突出，提高区分度。

#### 回退机制

每帧结果决策逻辑：

```csharp
// 优先使用共振峰分类（带置信度与平滑）
int raw = _detectedVowel[0];
float rawConf = _detectedConfidence[0];
_smoothedConfidence = Mathf.Lerp(_smoothedConfidence, rawConf, 0.5f);

if (raw >= 0 && _smoothedConfidence > 0.22f)
{
    pos = raw;  // 高置信度：采用共振峰结果
}
else
{
    // 回退：能量频带法 + 帧间平滑
    for (int i = 0; i < n; i++)
    {
        peaks[i] = Mathf.Lerp(_previousPeakValues[i], _peakValue[i], 0.6f);
        _previousPeakValues[i] = peaks[i];
        totalMouthWeight += peaks[i];
    }
    // 归一化后取最大值
    pos = argmax(normalizedPeaks);
}
```

**调优建议：**

- 在不同说话者/音量/噪声条件下实测调整 `means` 参考均值和 `f1Sigma`/`f2Sigma` 标准差
- 置信度阈值 `0.22f` 和 EMA 系数可根据识别效果微调
- 滤波器大小 `FILTER_SIZE=7` 和方差 `FILTER_DEVIATION_SQUARE=5.0` 影响平滑程度
- `PowOfAEIOU=1.2` 用于增强能量法中各元音间的差异
