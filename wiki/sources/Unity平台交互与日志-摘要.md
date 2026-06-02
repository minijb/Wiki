---
title: "Unity 平台交互与日志 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/gamedev/gameplay/unity-platform-and-logging.md"
tags: [unity, platform, android, ios, logging, network]
---

# Unity 平台交互与日志

## 来源

`raw/gamedev/gameplay/unity-platform-and-logging.md` — 合并自 Unity 平台接入与日志系统笔记

## 要点

1. **平台交互总览** — Android 接入两种方式（AndroidJavaClass / Jar插件）、UnityPlayerActivity 自定义、C#↔Java 互调（AndroidJavaClass/AndroidJNI + UnitySendMessage/AndroidJavaProxy）、iOS 交互（.m/.mm 插件 + DllImport）
2. **Android 详解** — NDK(.so) vs SDK(.aar) 两种底层方式、C# 调用 Java 四种示例（静态/非静态/Activity静态/Activity实例）、必须在 Start() 中初始化 AndroidJavaObject、SDK/JDK/Gradle 版本配置
3. **iOS 详解** — Plugins/iOS/ 目录、DllImport("__Internal")、回调：UnitySendMessage vs AOT（MonoPInvokeCallback 委托）、iOS 端 extern "C" 导出函数结构
4. **网络同步** — 客户端-服务器时间差处理：双端时间表 + 延迟补偿策略
5. **日志基础** — Debug.Log/Warning/Error、关联 GameObject 定位、彩色日志（rich text color）
6. **日志本地存储** — 基础实现（logMessageReceived + File）、生产级多线程实现（logMessageReceivedThreaded + ConcurrentQueue + ManualResetEvent + StreamWriter 后台写线程）
7. **日志上传** — UnityWebRequest + WWWForm + AddBinaryData 上传日志文件
8. **条件编译** — `[Conditional("OPEN_LOG")]` 控制日志代码编译、ScriptingDefineSymbols 编辑器宏管理、全平台批量设置

## 关联 Wiki 页面

- [[concepts/CSharp网络Socket|C# Socket 网络]]
- [[concepts/CSharp文件IO|C# 文件 IO]]
- [[concepts/CSharp并发模型|C# 并发模型]]
- [[concepts/Unity编辑器全局设置|编辑器全局设置]]
