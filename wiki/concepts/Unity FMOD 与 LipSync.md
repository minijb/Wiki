---
title: "Unity FMOD 与 LipSync"
type: concept
updated: 2026-05-11
tags: [unity, audio, fmod, lipsync, dsp]
---

# Unity FMOD 与 LipSync

FMOD 是 Unity 中广泛使用的专业音频中间件，提供 Event、Bus（三层总线架构）、Parameter、VCA、Snapshot 等完整音频管理能力，支持 DSP 数字信号处理与实时频谱分析。在此基础上构建的 LipSync 唇形同步系统，采用共振峰估计（Formant Estimation）结合高斯相似度分类，并利用 Unity JobSystem + Burst 实现高性能并行处理，根据实时音频驱动角色口型动画。

## FMOD 音频系统

FMOD 音频系统以 Event 为核心播放单元，围绕 Bus 总线体系进行信号路由与混音，通过 Parameter、VCA、Snapshot 等机制实现动态音频控制。

### Event 事件

Event 是 FMOD Studio 设计好的音频资源基本播放单元，分为功能性事件（暂停/恢复等控制）和音频类事件（常规声音播放）。`EventInstance` 生命周期为：

```text
CreateInstance → start → release → 异步销毁
```

`start()` 后立即调用 `release()` 标记可释放，异步更新系统在 `FMOD_STUDIO_PLAYBACK_STOPPED` 状态下自动销毁实例。多次触发同一事件可创建新 `EventInstance` 或使用 `step` 停止后重播。

### 参数系统

FMOD 提供两种参数类型：

| 参数类型 | 作用范围 | 调用方式 |
|:---------|:---------|:---------|
| 事件参数 | 特定事件实例 | `eventInstance.setParameterByName("参数名", 值)` |
| 全局参数 | 所有事件共享 | `RuntimeManager.StudioSystem.setParameterByName("参数名", 值)` |

事件参数适合单个音效的动态变化（如脚步音随地形材质变化）；全局参数适合影响整个游戏环境（天气、战斗状态、情绪）。

### VCA 音量控制

VCA（Voltage Controlled Amplifier）是纯音量推子，**不支持 DSP 特效**，仅做音量乘法运算。核心优势在于**跨层控制**：FMOD Bus 是树状结构，设置 `bus:/SFX` 仅影响自身与子 Bus，而 VCA 可将任意层级、不同分支的 Bus 绑定在一起统一控制音量。

| 对比 | Bus | VCA |
|:-----|:---|:----|
| 控制内容 | 音频信号流、混音、DSP | 仅音量 |
| DSP 支持 | ✅ | ❌ |
| 层级结构 | 树状嵌套 | 平级无嵌套 |
| 跨层控制 | 仅自身及子 Bus | 可跨任意层级 |

VCA 是游戏 UI 设置中"音乐/语音/音效"音量滑块的首选方案。

### 快照与延音点

**快照（Snapshot）** 是动态混音预设，可在不同游戏状态（战斗、洞穴、水下、暂停）下同时修改多个 Bus 的音量和效果参数。不创建新 Bus，仅调整已有 Bus 参数；多个快照可同时生效并互相过渡。

**延音点（Sustain Point）** 配合 Loop 区实现 `[Intro] → [Loop] → [Release]` 三段式播放。播放到延音点时暂停/循环，直到 `keyOff()` 调用后继续播放 Release 结尾段。适用于蓄力技能音效、音乐结构控制、剧情演出音效等渐进/节奏性过渡场景。

### 事件回调

当 `EventInstance` 销毁时触发 `EVENT_CALLBACK_TYPE.DESTROYED` 回调，可用于清理资源、日志记录。回调在音频线程执行，**不能直接调用 Unity API**（如 `GameObject`、`Transform`），需通过线程安全队列中转。

### Master Bank 与 Strings Bank

Master Bank 包含全局混音器（Mixer）、快照和全局参数，是 FMOD 项目的核心 Bank —— 即使事件的 `.bank` 已加载，Master Bank 未加载则事件无法播放。Strings Bank 存储事件/Bus/参数路径到 GUID 的运行时映射，使开发者可用字符串访问事件，本身不含音频数据，不影响内存占用。

## Bus 总线体系

FMOD 提供三层 Bus 架构，形成完整的音频信号路由链路：

```text
[Event] → [Group Bus] → [Master Bus] → 音频输出
              ↓ Send
         [Return Bus] ────↗
```

### Master Bus（主总线）

所有音频最终的输出节点，完成全局混音后输出到音频设备。**不可跳过**，控制全局音量、动态范围压缩、全局静音/暂停。

### Group Bus（分组总线）

分类管理不同类型音效，允许单独调整各类别的音量、混响、EQ：

| Bus | 用途 |
|:----|:-----|
| Music Bus | 背景音乐 |
| SFX Bus | 音效（UI、环境、战斗） |
| Voice Bus | 角色语音/对话 |

### Return Bus（返回总线）

接收多个 Group Bus 的 Send 信号，统一处理 DSP 特效后将结果返回 Master Bus。音效不直接流入 Return Bus，而是由 Group Bus 通过 Send 发送部分信号。**多个 Group Bus 共享同一 Return Bus 可节省 CPU 计算量**。典型场景：洞穴/教堂混响、水下低通滤波、全局特效处理。

> [!warning] Bus 串联
> Bus 是串联连接，路径上所有 Bus 必须保持闭合状态，否则链路中断。

## DSP 数字信号处理

FMOD 提供 DSP（Digital Signal Processor）系统，允许将数字信号处理器挂载到音频通道上，实现实时频谱分析、音效修改、数据监控等功能。

### DSP 生命周期

完整的 DSP 使用流程：

```text
createDSPByType → setParameterInt → flushCommands → addDSP → 使用 → removeDSP → release
```

关键点：`setParameterXXX` 调用后必须执行 `flushCommands()` 使参数生效；清理时必须**先 `removeDSP` 再 `release`**，顺序不可颠倒。

### ChannelGroup

DSP 通过 `ChannelGroup` 附着到音频事件。通过 `eventInstance.getChannelGroup()` 获取通道组后，使用 `channelGroup.addDSP(index, dsp)` 挂载 DSP。

### FFT 频谱分析

使用 `DSP_TYPE.FFT` 创建频谱分析器，配置参数：

- **窗函数类型**（`DSP_FFT.WINDOW`）：影响频谱泄漏
- **窗口大小**（`DSP_FFT.WINDOWSIZE`）：2048/4096，乘 2 表示双倍精度

通过 `dsp.getParameterData(DSP_FFT.SPECTRUMDATA, out data, out length)` 获取数据指针，`Marshal.PtrToStructure` 还原为 `DSP_PARAMETER_FFT` 结构体，调用 `getSpectrum(channelIndex, ref spectrumArray)` 提取具体通道频谱。

DSP 系统是构建音频驱动功能（频谱可视化、节拍检测、唇形同步）的核心基础设施。

## LipSync 唇形同步

唇形同步技术根据音频输入实时检测当前发音的元音，驱动角色的口型动画。本方案采用**共振峰估计（Formant Estimation）** 结合**高斯相似度分类**，利用 Unity JobSystem + Burst 实现高性能并行处理。

### 技术栈总览

| 技术层 | 方法 | 目的 |
|:-------|:-----|:-----|
| 频谱预处理 | 高斯滤波卷积 + 帧内归一化 | 平滑频谱，减小能量波动影响 |
| 能量过滤 | EMA 自适应静音阈值 | 丢弃低能量帧，减少误识别 |
| 帧间平滑 | 指数平滑（能量/置信度） | 降低抖动，避免口型突变 |
| 核心分类 | 共振峰估计 (F1/F2) + 高斯相似度 | 区分 A/E/I/O/U 五个元音 |
| 回退机制 | 能量频带法 | 低置信度时回退到能量分布判定 |
| 并行加速 | Unity JobSystem + Burst | 卷积、归一化、能量计算、分类并行 |

### 信号处理管线

```text
原始 Spectrum
    │
    ▼
[高斯卷积平滑] ─── IJobParallelFor ─── 每个频点独立卷积
    │
    ▼
[帧内归一化] ─── IJob ─── 除以最大值，消除能量波动
    │
    ▼
[语音清晰度评分] ─── IJob ─── 300–3400Hz 频段能量占比
    │
    ├── [共振峰分类] IJob ─── EstimateFormants + ClassifyVowelByFormants
    │
    └── [能量频带法] IJobParallelFor ─── 每个元音独立计算加权频带能量
    │
    ▼
[置信度判断]
    ├── confidence > 0.22 → 采用共振峰结果
    └── confidence ≤ 0.22 → 回退能量法 + 帧间指数平滑
    │
    ▼
[输出元音索引] ─── 驱动 BlendShape / 骨骼动画
```

### 共振峰估计

从平滑后的频谱中寻找前两个局部峰值（`spectrum[i] > spectrum[i-1] && spectrum[i] > spectrum[i+1]`），平方加权增强高能量峰区分度，取最大的两个作为第一共振峰（F1）和第二共振峰（F2）。索引到频率转换：`freq = index × (sampleRate / 2) / spectrumLength`。

### 高斯相似度分类

基于 F1/F2 估计值，计算与五个标准元音参考均值的二维高斯相似度：

| 元音 | F1 (Hz) | F2 (Hz) | 特征 |
|:-----|:--------|:--------|:-----|
| A | 800 | 1200 | 开口最大，F1 高 |
| E | 400 | 2000 | F2 较高，F1 中等偏低 |
| I | 300 | 2400 | F1 最低，F2 最高 |
| O | 500 | 1000 | F1/F2 均中等 |
| U | 350 | 800 | F1/F2 均较低 |

计算各元音的马氏距离 `d1 = (F1 - μ_F1) / σ_F1`，`d2 = (F2 - μ_F2) / σ_F2`，应用二维高斯核 `exp(-0.5×(d1²+d2²))`，Softmax 归一化后取最高分作为分类结果，归一化得分作为置信度。

### JobSystem 并行化

六类 Burst 编译 Job 构成完整管线：

| Job | 类型 | 职责 |
|:----|:-----|:-----|
| `GetSmoothSpectrum` | `IJobParallelFor` | 每个频点独立高斯卷积 |
| `NormalizeSpectrum` | `IJob` | 全频谱除以最大值归一化 |
| `GetClarity` | `IJob` | 计算 300–3400Hz 语音频段占比 |
| `LipSyncJob` | `IJobParallelFor` | 每个元音并行计算加权频带能量 |
| `FormantClassifyJob` | `IJob` | 共振峰估计与高斯分类 |

调度策略：`GetSmoothSpectrum → NormalizeSpectrum → GetClarity` 顺序依赖；`LipSyncJob` 与 `FormantClassifyJob` 在 `GetClarity` 后**并行执行**，通过 `JobHandle.CombineDependencies` 合并等待。每帧 `Allocate(TempJob) → Schedule → Complete → Dispose` 管理 NativeArray 生命周期，零内存泄漏。

### 回退与平滑机制

每帧结果决策：共振峰置信度 `> 0.22`（经帧间平滑后）采用共振峰结果；否则回退到能量频带法。两种路径均做帧间指数平滑：

- 能量法 peak 值：`Lerp(prev, cur, 0.6)`
- 共振峰置信度：`Lerp(prev, cur, 0.5)`

EMA 自适应静音阈值：每帧维护频谱能量和的指数移动平均（`α = 0.2`），自适应阈值为 `max(baseThreshold, EMA × 0.35)`，低于阈值的帧丢弃，避免静音段误识别。

## 参见

- [[sources/Unity-FMOD-LipSync-摘要|来源摘要]] — 原始文件要点
