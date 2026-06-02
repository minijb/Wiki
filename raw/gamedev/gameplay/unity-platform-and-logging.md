---
title: Unity 平台交互与日志系统
type: source
updated: 2026-06-02
tags:
  - unity
  - platform
  - android
  - ios
  - logging
---

# Unity 平台交互与日志系统

> 合并自 Unity 平台接入笔记、Android/iOS 交互实践、日志系统与面试精华。

---

## 一、平台交互总览

### Android 接入 Unity 的两种方式

1. **AndroidJavaClass / AndroidJavaObject**：通过 JNI 调用 Android 系统方法，无需 Android 资源包，适合简单调用
2. **Jar/AAR 插件**：将 Java 代码作为插件导入 Unity，可接入任意 Java 库，灵活性高

### UnityPlayerActivity

Unity 在 Android 平台的核心 Activity，负责启动 Unity 引擎、管理生命周期、处理与 Android 系统的交互。可通过自定义继承 `UnityPlayerActivity` 的 Activity 实现 Unity 与 Android 的双向交互。

**获取 class.jar：**
- Build Settings 中勾选 `Export Project` 导出 Android Studio 项目
- 或从编辑器安装目录中查找 `unity-classes.jar`

### C# ↔ Java 互调方式

**C# 调用 Java：**
- `AndroidJavaClass` / `AndroidJavaObject`：高层封装
- `AndroidJNI` / `AndroidJNIHelper`：底层 JNI 调用

**Java 调用 C#：**
- `UnitySendMessage(GameObject, MethodName, Parameter)`
- `AndroidJavaProxy`：通过接口回调

### Unity ↔ iOS 交互方式

**C 文件 (.m / .mm)：** 放到 `Plugins/iOS/` 目录下。

```csharp
[DllImport("__Internal")]
private static extern void IOSShowDialog(string title, string message,
    string confirmButton, string cancelButton,
    SuccessCallback success, FailCallback fail);
```

**回调方式：**
- `UnitySendMessage`：需要指定 GameObject 且必须 Enable
- AOT 回调（推荐）：使用委托 + `MonoPInvokeCallback` 属性

```csharp
private delegate void SuccessCallback();
private delegate void FailCallback();

[MonoPInvokeCallback(typeof(SuccessCallback))]
private static void OnSuccess() { /* ... */ }

[MonoPInvokeCallback(typeof(FailCallback))]
private static void OnFail() { /* ... */ }
```

**iOS 端 C++ 结构：**

```cpp
extern "C" {
    typedef void (*SuccessCallback)();
    typedef void (*FailCallback)();
    void IOSShowDialog(const char* title, const char* message,
        const char* confirm, const char* cancel,
        SuccessCallback success, FailCallback fail);
}
```

---

## 二、Android 交互详解

### 两种底层交互方式

| 方式 | 文件格式 | 调用方式 | 特点 |
|------|---------|---------|------|
| NDK (C++) | .so | `[DllImport]` | 高性能，适合计算密集型 |
| Java SDK | .aar / .jar | `AndroidJavaClass/Object/Proxy` | 可调用 Android 高级 API |

### C# 调用 Java 示例

**1. 调用普通类的静态方法：**

```csharp
AndroidJavaObject helper = new AndroidJavaObject("pers.study.android2unity.Helper");
helper.CallStatic("getMessageFromUnity", "我是 unity");
```

**2. 调用普通类的非静态方法：**

```csharp
AndroidJavaObject helper = new AndroidJavaObject("pers.study.android2unity.Helper");
helper.Call("setAndroidForUnityListener", listener);
```

**3. 调用继承 UnityPlayerActivity 的 Activity（静态方法）：**

```csharp
AndroidJavaClass jc = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
AndroidJavaObject context = jc.GetStatic<AndroidJavaObject>("currentActivity");
AndroidJavaClass bridge = new AndroidJavaClass("com.bridge.BridgeActivity");
bridge.CallStatic("showToast", context);
```

**4. 调用 Activity 实例方法：**

```csharp
AndroidJavaClass jc = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
AndroidJavaObject context = jc.GetStatic<AndroidJavaObject>("currentActivity");
context.Call("add", 15, 9);
```

> **注意：** `AndroidJavaClass`/`AndroidJavaObject` 必须在 `Start()` 中进行初始化，默认的 `new` 无法获取 Java Object。

### SDK / JDK / Gradle 版本

- [Android SDK 版本参考](https://docs.unity.com/zh-cn/build-automation/reference/available-android-sdk-versions)
- [JDK 版本说明](https://docs.unity3d.com/cn/2023.2/Manual/android-sdksetup.html)

**常见问题：**
- Build 时 Gradle 版本不匹配 → `Project Settings → Other Settings → Target API Level` 中降低 API Level
- 使用 `AndroidJavaClass`/`AndroidJavaObject` 需在 `Start()` 中初始化

---

## 三、iOS 交互详解

### 文件放置

- `.m` / `.mm` 文件放到 `Plugins/iOS/` 目录
- Unity Build 时会自动包含这些文件

### 调用方法

```csharp
[DllImport("__Internal")]
private static extern void IOSNativeMethod(string param);

[DllImport("__Internal")]
private static extern void IOSDoVibrate(long milliseconds);

[DllImport("__Internal")]
private static extern void IOSShowToast(string message);
```

### 回调实现 (AOT)

```csharp
// C# 端
private delegate void SuccessCallback();
private delegate void FailCallback();

[MonoPInvokeCallback(typeof(SuccessCallback))]
private static void OnSuccessCallback() { }

[MonoPInvokeCallback(typeof(FailCallback))]
private static void OnFailCallback() { }

[DllImport("__Internal")]
private static extern void IOSShowDialog(string title, string message,
    string confirm, string cancel,
    SuccessCallback success, FailCallback fail);
```

```cpp
// C/C++ 端 (Plugins/iOS/NativeBridge.mm)
extern "C" {
    typedef void (*SuccessCallback)();
    typedef void (*FailCallback)();
    void IOSShowDialog(const char* title, const char* message,
        const char* confirm, const char* cancel,
        SuccessCallback success, FailCallback fail) {
        // 创建 UIAlertController 并调用回调
    }
}
```

### iOS 回调注意事项

- `UnitySendMessage` 依赖具体的 GameObject 名称，且该对象必须 Enable
- AOT 回调方案更解耦、更可靠
- iOS 编译需要 `--enable-il2cpp`，委托需标记 `[MonoPInvokeCallback]`

---

## 四、网络同步策略

### 客户端-服务器时间差处理

核心思路：客户端和服务器各维护一个技能时间表。

**客户端侧：**
- 技能释放时，检查本地时间列表
- 若时间已到且列表中有该技能信息 → 立即释放
- 否则等待

**服务器侧：**
- 下发技能信息时附带服务器时间戳
- 客户端收到后：
  - 若本地时间已过服务器时间 → 立即释放
  - 否则存入列表，等待本地时间到达后自动释放

> 本质是双端时间表 + 延迟补偿，保证客户端即使在时间差下也能正确执行技能。

---

## 五、日志系统

### 基础打印

```csharp
Debug.Log("This is a log message.");
Debug.LogWarning("This is a warning message!");
Debug.LogError("This is an error message!");

// 点击日志定位 GameObject（可指向 Prefab）
GameObject go = new GameObject("go");
Debug.Log("Test", go);  // 第二个参数关联对象
```

### 彩色日志

```csharp
Debug.LogFormat("This is <color=#ff0000>{0}</color>", "red");
Debug.LogFormat("This is <color=#00ff00>{0}</color>", "green");
Debug.LogFormat("This is <color=#0000ff>{0}</color>", "blue");
Debug.LogFormat("This is <color=yellow>{0}</color>", "yellow");
```

### 本地文件存储

**基础实现：**

```csharp
void Awake() {
    var t = DateTime.Now.ToString("yyyyMMddhhmmss");
    string path = $"{Application.persistentDataPath}/output_{t}.log";
    Application.logMessageReceived += OnLogCallback;
}

void OnLogCallback(string condition, string stackTrace, LogType type) {
    m_logStr.Append(condition + "\n" + stackTrace + "\n");
    if (m_logStr.Length <= 0) return;
    File.AppendAllText(m_logFileSavePath, m_logStr.ToString());
    m_logStr.Clear();
}
```

**多线程安全实现（生产级）：**

```csharp
public class LogHelper : MonoBehaviour {
    private StreamWriter mStreamWriter;
    private ConcurrentQueue<LogData> mQueue = new();
    private ManualResetEvent mSignal = new(false);
    private bool mRunning = false;

    public void Init(string savePath, string logFileName) {
        mStreamWriter = new StreamWriter(Path.Combine(savePath, logFileName));
        Application.logMessageReceivedThreaded += OnLogThreaded;
        mRunning = true;
        new Thread(FileLogThread).Start();
    }

    void OnLogThreaded(string condition, string stackTrace, LogType type) {
        mQueue.Enqueue(new LogData {
            log = $"{DateTime.Now:yyyy:MM:dd HH:mm:ss} {condition}",
            trace = stackTrace, type = type
        });
        mSignal.Set();  // 通知写线程
    }

    void FileLogThread() {
        while (mRunning) {
            mSignal.WaitOne();
            if (mStreamWriter == null) break;
            while (mQueue.TryDequeue(out LogData data)) {
                mStreamWriter.Write($"{data.type} >>> {data.log}\n{data.trace}\n\r\n");
            }
            mStreamWriter.Flush();
            mSignal.Reset();
            Thread.Sleep(1);
        }
    }

    void OnApplicationQuit() {
        Application.logMessageReceivedThreaded -= OnLogThreaded;
        mRunning = false;
        mSignal.Set();
        mStreamWriter?.Close();
    }
}
```

**关键点：**
- `Application.logMessageReceivedThreaded`：在非主线程回调，适合生产者-消费者模式
- `ConcurrentQueue`：无锁队列，生产线程写入，消费线程读取
- `ManualResetEvent`：线程间信号通知，避免空转
- `Flush`：确保流数据落盘

### 日志上传

```csharp
IEnumerator UploadLog(string url, string logFilePath) {
    byte[] data = File.ReadAllBytes(logFilePath);
    WWWForm form = new WWWForm();
    form.AddField("desc", "test upload log file");
    form.AddBinaryData("logfile", data, "log.txt", "application/x-gzip");
    UnityWebRequest request = UnityWebRequest.Post(url, form);
    yield return request.SendWebRequest();
    if (!string.IsNullOrEmpty(request.error))
        Debug.LogError(request.error);
    else
        Debug.Log("上传完毕: " + request.downloadHandler.text);
}
```

### 条件编译控制

```csharp
[Conditional("OPEN_LOG")]
public static void InitLog(LogConfig cfg = null) { /* ... */ }

[Conditional("OPEN_LOG")]
public static void LogGreen(object obj) { ColorLog(obj, LogColor.Green); }

[Conditional("OPEN_LOG")]
public static void LogRed(object obj) { ColorLog(obj, LogColor.Red); }
```

通过 `ScriptingDefineSymbols` 在编辑器中控制宏开关：

```csharp
// 开启
ScriptingDefineSymbols.AddScriptingDefineSymbol("OPEN_LOG");
// 关闭
ScriptingDefineSymbols.RemoveScriptingDefineSymbol("OPEN_LOG");
```

支持全平台批量设置（Standalone / iOS / Android）。只有开启对应宏时，`[Conditional]` 方法及其调用才会被编译进去。

### Editor 中自动添加日志系统

```csharp
[MenuItem("ZMLog/打开日志系统")]
public static void LoadReport() {
    ScriptingDefineSymbols.AddScriptingDefineSymbol("OPEN_LOG");
    GameObject reportObj = Instantiate(
        AssetDatabase.LoadAssetAtPath<GameObject>("Assets/.../Reporter.prefab"));
    reportObj.name = "Reporter";
    AssetDatabase.SaveAssets();
    EditorSceneManager.SaveScene(EditorSceneManager.GetActiveScene());
    AssetDatabase.Refresh();
}
```

---

## 六、面试精华

### 多相机叠加渲染

- 多相机同时激活 → 叠加渲染
- `depth` 控制渲染顺序
- `Depth Only`：先渲染 depth 小的相机内容作为背景，再渲染 depth 大的相机。新帧画面根据深度信息与旧画面融合
- 限定范围使用 RenderTexture

### 纹理压缩格式

ERC 和 ASTC — 移动平台常用压缩格式。

### AB 包压缩

- **LZ4**：块压缩，压缩率低但读取快，支持按需加载
- **LZMA**：流压缩，压缩率高但只能流式读取

### AssetBundle.Unload

- `false`：只断开索引与资源的连接，不销毁资源。再次加载引入新资源，可能冗余
- `true`：完全销毁，可能丢失引用。通常切换场景时使用

### PlayerPrefs / JSON / 二进制

| 方案 | 特点 | 适用场景 |
|------|------|---------|
| PlayerPrefs | 键值对，轻量 | 简单配置 |
| JSON | 字符串，可读 | 网络传输、可读配置 |
| 二进制 | 体积最小 | 大量数据、网络协议 |

---

## 参考资源

- [Unity Android SDK 版本](https://docs.unity.com/zh-cn/build-automation/reference/available-android-sdk-versions)
- [Unity JDK 设置](https://docs.unity3d.com/cn/2023.2/Manual/android-sdksetup.html)
- [UnityPlayerActivity 详解](https://blog.csdn.net/qq_33060405/article/details/147198174)
- [UnityWebRequest 文档](https://docs.unity.cn/cn/2023.2/ScriptReference/Networking.UnityWebRequest.html)
- [日志存储参考](https://www.yxtown.com/my/course/70)
