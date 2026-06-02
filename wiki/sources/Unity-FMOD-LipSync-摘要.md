---
title: "Unity FMOD 音频与 LipSync — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/audio/unity-fmod-lipsync.md"
tags: [unity, audio, fmod, lipsync, dsp]
---

# Unity FMOD 音频与 LipSync

## 来源

`raw/gamedev/audio/unity-fmod-lipsync.md` — Unity FMOD 音频中间件与 LipSync 唇形同步技术的系统梳理，涵盖 Event 生命周期、Bus 总线体系、DSP 数字信号处理、共振峰唇形同步管线及 Unity JobSystem 并行化

## 要点

1. **Event 事件体系** — Event 是 FMOD 音频资源的基本播放单元，分为功能性事件（暂停/恢复等控制）和音频类事件（常规播放）。`EventInstance` 生命周期为 `CreateInstance → start → release → 异步销毁`，`release()` 后异步系统在 `FMOD_STUDIO_PLAYBACK_STOPPED` 状态自动回收
2. **Bus 三层总线架构** — Master Bus 是最终输出节点，不可跳过；Group Bus 分类管理（Music/SFX/Voice）；Return Bus 接收多 Group Bus 的 Send 信号统一处理 DSP 特效并返回 Master Bus。Bus 串联连接，路径上所有 Bus 必须闭合
3. **参数系统** — 事件参数（`setParameterByName`）控制单个音效表现（如脚步随地形的变化）；全局参数（`setParameterByName` on StudioSystem）跨事件共享（天气、战斗状态）。支持 `setParameterByNameWithLabel` 使用 FMOD Studio 中定义的标签值
4. **VCA 音量控制** — VCA 是纯音量推子，可跨 Bus 层级控制多个不相关 Bus 的音量，不支持 DSP。相比 Bus 的树状结构，VCA 平级无嵌套，性能开销极小（仅做音量乘法），是 UI 设置中音量滑块的首选方案
5. **快照与延音点** — 快照（Snapshot）是动态混音预设，可同时调整多个 Bus 参数实现战斗/洞穴/水下等场景音频切换。延音点（Sustain Point）配合 Loop 区实现 `[Intro] → [Loop] → [Release]` 播放结构，`keyOff()` 触发跳出循环
6. **Master Bank 与 Strings Bank** — Master Bank 包含全局混音器、快照和全局参数，所有 EventInstance 依赖它。Strings Bank 存储事件/Bus/参数路径到 GUID 的映射，使开发者可用字符串访问事件
7. **DSP 数字信号处理** — DSP 通过 ChannelGroup 挂载到音频通道。完整生命周期：`createDSPByType → setParameterInt → flushCommands → channelGroup.addDSP → 使用 → removeDSP → dsp.release`。`flushCommands()` 是 `setParameter` 调用生效的前提
8. **FFT 频谱分析** — 使用 `DSP_TYPE.FFT` 创建频谱分析器，配置窗函数类型和窗口大小后，通过 `getParameterData(DSP_FFT.SPECTRUMDATA)` 获取 `DSP_PARAMETER_FFT` 结构体，再调用 `getSpectrum(channelIndex, ref spectrumArray)` 提取实时频谱数据
9. **LipSync 共振峰管线** — 处理流程：高斯卷积平滑 → 帧内归一化（除以最大值消除能量波动）→ 清晰度评分（300–3400Hz 频段占比）→ 并行分支：共振峰估计（寻找前两个局部峰值作为 F1/F2）与能量频带法（每个元音独立计算加权频带能量）。共振峰分类使用二维高斯核 `exp(-0.5×(d1²+d2²))` 计算到五个元音参考均值（A/E/I/O/U）的马氏距离，Softmax 归一化后取最高分
10. **JobSystem 并行化** — 六个 Burst 编译 Job 构成管线：`GetSmoothSpectrum`（`IJobParallelFor` 卷积）、`NormalizeSpectrum`（`IJob` 全频谱归一化）、`GetClarity`（`IJob` 语音清晰度）、`LipSyncJob`（`IJobParallelFor` 能量法）与 `FormantClassifyJob`（`IJob` 共振峰分类）在 `GetClarity` 后并行执行，通过 `JobHandle.CombineDependencies` 合并等待。每帧 `Allocate(TempJob) → Schedule → Complete → Dispose` 管理 NativeArray 生命周期
11. **置信度与回退机制** — 共振峰结果置信度 `> 0.22` 时直接采用；低于阈值时回退到能量频带法。两路结果均做帧间指数平滑（peak 值 `Lerp(prev, cur, 0.6)`，置信度 `Lerp(prev, cur, 0.5)`）防止口型抖动。EMA 自适应静音阈值（`α=0.2`）动态追踪能量趋势，过滤低能量帧

## 关联 Wiki 页面

- [[concepts/Unity FMOD 与 LipSync|Unity FMOD 与 LipSync]] — 概念页
