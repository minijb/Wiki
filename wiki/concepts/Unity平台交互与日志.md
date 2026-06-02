---
title: "Unity 平台交互与日志"
type: concept
updated: 2026-06-02
tags: [unity, platform, android, ios, logging, network]
---

# Unity 平台交互与日志

Unity 跨平台开发中的原生交互（Android/iOS）和日志系统的实践经验。

## 平台交互架构

Unity 通过 JNI（Android）和 P/Invoke（iOS）实现与原生平台的通信。

### Android

**两种接入策略：**

| 方式 | 文件 | 适用场景 |
|------|------|---------|
| `AndroidJavaClass`/`AndroidJavaObject` | 无 | 简单系统调用 |
| Jar/AAR 插件 | .jar/.aar | 复杂 SDK 集成 |

**底层交互：**
- NDK (.so) + `DllImport`：C/C++ 高性能
- Java SDK (.aar) + `AndroidJavaClass/Object/Proxy`：调用 Android 高级 API

**关键 API：**

```csharp
// 获取当前 Activity
AndroidJavaClass jc = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
AndroidJavaObject activity = jc.GetStatic<AndroidJavaObject>("currentActivity");

// 调用方法
activity.Call("methodName", args);
```

> `AndroidJavaObject` 必须在 `Start()` 中初始化，默认 `new` 无法获取 Java 对象。

### iOS

通过 `.m`/`.mm` 文件（放入 `Plugins/iOS/`）实现原生调用：

```csharp
[DllImport("__Internal")]
private static extern void IOSNativeMethod(string param);
```

**回调方案对比：**

| 方案 | 机制 | 要求 |
|------|------|------|
| `UnitySendMessage` | GameObject 名称 + 方法名 | 对象必须 Enable |
| AOT 委托 | `MonoPInvokeCallback` + delegate | IL2CPP 兼容 |

AOT 回调更解耦、更可靠，是生产推荐方案。

## 网络同步

### 客户端-服务器时间差处理

核心思路：双端维护时间表。

- 客户端技能释放时检查本地时间表
- 服务器下发信息附带时间戳
- 客户端收到信息后：若本地时间已过 → 立即执行；否则排队等待

本质是双端时间表 + 延迟补偿，保证时间差下的正确执行。

## 日志系统

### 基础用法

```csharp
Debug.Log("message");           // 普通日志
Debug.LogWarning("warning");    // 警告
Debug.LogError("error");        // 错误
Debug.Log("Test", gameObject);  // 关联对象，点击可定位
```

彩色日志：`<color=#ff0000>红色文本</color>`

### 本地存储

**多线程安全方案（生产级）：**

```
[主线程/回调线程] ──Enqueue──→ [ConcurrentQueue<LogData>]
                                       │
                              ManualResetEvent.Set()
                                       │
                              [后台写线程] ──→ StreamWriter → 文件
```

- `Application.logMessageReceivedThreaded`：多线程安全的日志回调
- `ConcurrentQueue`：无锁生产者-消费者队列
- `ManualResetEvent`：避免写线程空转
- `StreamWriter.Flush()`：确保落盘

### 条件编译

```csharp
[Conditional("OPEN_LOG")]
public static void Log(string msg) { /* ... */ }
```

通过 `ScriptingDefineSymbols` 在编辑器中控制，全平台（Standalone/iOS/Android）批量管理宏定义。未开启时，方法体及所有调用点都不会编译。

### 日志上传

使用 `UnityWebRequest.Post` + `WWWForm.AddBinaryData` 上传日志文件到服务器。

## 资源管理补充

- `AssetBundle.Unload(false)`：断开引用不销毁资源；`Unload(true)`：完全销毁
- **LZ4**：块压缩，读取快；**LZMA**：流压缩，压缩率高
- 移动端纹理压缩：ERC / ASTC

## 参见

- [[concepts/Unity常用API速查|Unity 常用 API 速查]] — Transform/UI/编辑器扩展
- [[concepts/CSharp网络Socket|C# Socket 网络]] — 网络编程基础
- [[concepts/CSharp文件IO|C# 文件 IO]] — 文件读写基础
- [[concepts/CSharp并发模型|C# 并发模型]] — 多线程与线程安全
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — 宏定义管理
- [[sources/Unity平台交互与日志-摘要|来源摘要]]
