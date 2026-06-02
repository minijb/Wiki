---
title: Unity 常用 API 速查表
type: source
updated: 2026-06-02
tags:
  - unity
  - api
  - cheatsheet
  - editor
  - ui
---

# Unity 常用 API 速查表

> 合并自 Unity 杂项笔记、CheatSheets、工具函数和面试精华。按类别组织，供日常开发速查。

---

## 一、Transform 与 GameObject

### 基本关系

- 两者都可以直接创建 GameObject，但不能相互转换
- 通过属性互相获取：`transform.gameObject` / `gameObject.transform`
- **频繁改 Transform** → 用 `Transform`；**常用 SetActive** → 用 `GameObject`
- **Transform 不能 new**，**GameObject 可以 new**；Transform 不能 Destroy，GameObject 可以

### 位移与方向

```csharp
// 手动设置位置
transform.position = transform.position + offset;

// API 方式
transform.Translate(Vector3.forward);                           // local space
transform.Translate(Vector3.forward, Space.World);              // world space
transform.Translate(transform.forward, Space.World);            // 自身坐标系在世界空间的朝向
```

- `Vector3.forward` 永远是 `(0, 0, 1)`
- `transform.forward` 是自身 Z 轴在世界坐标的指向，长度恒为 1，但方向不一定等于 `(0, 0, 1)`

### 父子关系

```csharp
transform.parent = null;                    // 解除父对象
transform.SetParent(other.transform);       // 设置父对象
transform.SetParent(other.transform, true); // 保持世界变换不变

transform.Find("ChildName");               // 查找子对象
transform.childCount;                      // 直接子对象数量（失活也算，不含孙子）
transform.GetChild(index);                  // 按索引获取子对象

// 子对象操作
child.IsChildOf(parent);                   // 判断是否子对象
child.GetSiblingIndex();                   // 获取兄弟序号
child.SetAsFirstSibling();                  // 移到兄弟最前
child.SetAsLastSibling();                   // 移到兄弟最后
child.SetSiblingIndex(index);              // 设置兄弟序号
```

---

## 二、Time 与帧率

```csharp
Time.timeScale = 1.0f;           // 时间缩放，0 暂停所有基于帧率的功能
Time.deltaTime;                   // 受 timeScale 影响的帧间隔
Time.unscaledDeltaTime;           // 不受 timeScale 影响的帧间隔
Time.fixedDeltaTime;              // 物理帧间隔
Time.fixedUnscaledDeltaTime;      // 不受 timeScale 影响的物理帧间隔
Time.frameCount;                  // 总帧数
Time.realtimeSinceStartup;        // 不受 timeScale 影响的真实时间
```

### TimeScale 关键点

- `timeScale` 为 0 时 `FixedUpdate` 完全停止，但 `Update`/`LateUpdate` 照常执行
- `timeScale` 为 0 时 `Coroutine` 中的 `WaitForSeconds`/`WaitForFixedUpdate` 会卡住，需用 `WaitForSecondsRealtime`
- 修改 `timeScale` 时推荐同步修改 `Time.fixedDeltaTime`

### 目标帧率

```csharp
Application.targetFrameRate = targetFrameRate;
```

---

## 三、输入系统

### 旧版 Input (Input Manager)

```csharp
Input.GetKey(KeyCode.W);        // 按住期间持续为 true
Input.GetKeyDown(KeyCode.W);    // 按下第一帧为 true，随后为 false
Input.GetKeyUp(KeyCode.W);      // 松开第一帧为 true
```

### 新版 Input System

**Action Type（触发时机）：**
- `Value`：值变化时触发，适合摇杆等连续输入
- `Button`：按下/按住/弹起时各触发一次
- `Pass Through`：按下和弹起时各触发一次

**Control Type**：指定绑定的输入设备类型。

```csharp
// 实时读取值
private PlayerInputAction playerInputAction;

void Awake() {
    playerInputAction = new PlayerInputAction();
    playerInputAction.Player.Enable();
}

public Vector2 GetMovementVectorNormalized() {
    Vector2 inputVector = playerInputAction.Player.Move.ReadValue<Vector2>();
    return inputVector.normalized;
}

// 事件订阅
void Awake() {
    playerInputAction = new PlayerInputAction();
    playerInputAction.Player.Enable();
    playerInputAction.Player.Interact.performed += Interact_Performed;
}

private void Interact_Performed(InputAction.CallbackContext obj) {
    OnInteractAction?.Invoke(this, EventArgs.Empty);
}
```

---

## 四、UI 系统

### 锚点与重心

- **Anchor（锚点）**：聚合状态下物体固定在父物体特定位置；分离状态下子物体适配父节点大小
- **Pivot（重心）**：旋转和缩放的参考点

### Canvas Scaler

`UI Scale Mode`：
- **Scale With Screen Size**：按参考分辨率缩放
- **Constant Pixel Size**：固定像素大小
- **Expand**：选择变化较小的方向放大，减少非等比缩放对布局的影响，适合小标准尺寸扩充到大屏
- **Shrink**：与 Expand 类似，适合缩小场景

### Content Size Fitter

自动根据子元素大小调整 UI 元素尺寸，确保内容不会截断或溢出。配合 `LayoutGroup` 使用可实现自动布局。

- `Horizontal Fit`：Unconstrained / Preferred Size / Min Size
- `Vertical Fit`：Unconstrained / Preferred Size / Min Size

### 判断点是否在 UI 范围内

**Camera 模式：**

```csharp
if (RectTransformUtility.ScreenPointToLocalPointInRectangle(
    rectTransform, touch.position, Camera.main, out Vector2 localPoint)) {
    if (rectTransform.rect.Contains(localPoint)) {
        Debug.Log("in area");
    }
}
```

**Overlay 模式（完美方案 — 世界坐标转换）：**

```csharp
private Rect GetWorldRect(RectTransform rectTransform) {
    Vector3[] corners = new Vector3[4];
    rectTransform.GetWorldCorners(corners);
    float width  = Math.Abs(Vector2.Distance(corners[0], corners[3]));
    float height = Math.Abs(Vector2.Distance(corners[0], corners[1]));
    return new Rect(corners[0], new Vector2(width, height));
}

private bool IsClickInArea(Vector2 position) {
    RectTransform rect = transform as RectTransform;
    if (rect == null) return false;
    Rect worldRect = GetWorldRect(rect);
    return position.x > worldRect.x && position.x < worldRect.x + worldRect.width
        && position.y > worldRect.y && position.y < worldRect.y + worldRect.height;
}
```

### ScreenPointToLocalPointInRectangle

```csharp
public static bool ScreenPointToLocalPointInRectangle(
    RectTransform rect, Vector2 screenPoint, Camera cam, out Vector2 localPoint);
```

- 将屏幕坐标转换为相对指定 RectTransform 的本地坐标
- Overlay 模式 Canvas 中 `cam` 参数传 `null`
- 常用于检测点击是否在 UI 上、计算两个 UI 相对位置

```csharp
// 计算两个 UI 的相对位置
public Vector2 CalculateUIRelativePos(RectTransform target, RectTransform arrow) {
    Vector2 pos = RectTransformUtility.WorldToScreenPoint(camera, target.position);
    RectTransformUtility.ScreenPointToLocalPointInRectangle(arrow, pos, camera, out Vector2 local);
    return local;
}
```

### UGUI 底层渲染

**顶点生成（OnPopulateMesh）：**

```csharp
protected virtual void OnPopulateMesh(VertexHelper vh) {
    var r = GetPixelAdjustedRect();
    var v = new Vector4(r.x, r.y, r.x + r.width, r.y + r.height);
    Color32 color32 = color;
    vh.Clear();
    vh.AddVert(new Vector3(v.x, v.y), color32, new Vector2(0f, 0f));
    vh.AddVert(new Vector3(v.x, v.w), color32, new Vector2(0f, 1f));
    vh.AddVert(new Vector3(v.z, v.w), color32, new Vector2(1f, 1f));
    vh.AddVert(new Vector3(v.z, v.y), color32, new Vector2(1f, 0f));
    vh.AddTriangle(0, 1, 2);
    vh.AddTriangle(2, 3, 0);
}
```

**Rebuild 流程：** `CanvasUpdateRegistry` 维护两个队列（LayoutRebuildQueue / GraphicRebuildQueue），在 `PerformUpdate` 中依次执行。仅发生变动时才将 UI 注册到队列。主要触发场景：
1. 顶点数据和材质参数更新
2. 层级关系变化
3. UI 大小变化
4. CanvasRenderer 的 Cull 模式变化

### OverDraw 优化

多次重绘同一像素造成的 GPU 开销，主犯是半透明物体（粒子+UI）：
- 对 Alpha 为 0 的 UI，勾选 CanvasRenderer 的 Cull Transparent Mesh，保持事件响应但不渲染
- 减少 Mesh 数量，多使用 RectMask2D

### 制作 Font

参考：[CSDN 文章](https://blog.csdn.net/zhunju0089/article/details/103125168)

---

## 五、序列化与 ScriptableObject

### SerializeField / Serializable

- `[SerializeField]`：将私有和保护字段暴露到 Inspector 并序列化
- `[Serializable]`：标记自定义类/结构体可序列化。**先对类 Serializable，再对字段 SerializeField**，才能在 Inspector 中显示

### ScriptableObject

用于存储可复用的数据资源，避免大量复制。

```csharp
[CreateAssetMenu(fileName = "TestObject", menuName = "ScritableObjects/TestObject")]
public class TestObject : ScriptableObject {
    public string MyString;
}
```

**用途：**
- 批量创建类型相同、内容不同的数据（武器数据、敌人配置等）
- 作为引用而非复制，节省内存
- 制作编辑器工具（如 URP 资源文件）

> **注意：** Build 后改变 ScriptableObject 的值会恢复为原始值。

### SerializedProperty

材质属性序列化后持久存储在资源中，即使删除属性，其序列化值和引用关系也不会丢失。命名相同时原来的值会自动赋值。

- 表示一种实例化关系，需配合 Editor 类使用
- 清理过期属性引用可减少不必要的资源保留

### PropertyAttribute / PropertyDrawer

- `PropertyAttribute`：用于标记字段
- `PropertyDrawer`：自定义属性在 Inspector 中的绘制方式
- 两者配合使用

### AutoHook 自动绑定

通过 CustomPropertyDrawer 实现：在 Inspector 中自动根据字段名查找子对象上以 `#字段名` 命名的 GameObject，并赋值对应组件。运行时字段名以 `#` 开头的子对象会被自动绑定。

---

## 六、常用模式与技巧

### 单例模式

**饿汉式（MonoBehaviour，Awake 初始化）：**

```csharp
public static WorldSoundFXManager instance;

private void Awake() {
    if (instance == null) {
        instance = this;
    } else {
        Destroy(gameObject);
    }
}
```

**懒汉式（非 MonoBehaviour，按需创建）：**

```csharp
public static ABManager GetInstance() {
    if (Instance == null) {
        Instance = new ABManager();
    }
    return Instance;
}
```

### 闭包与匿名函数的 GC 问题

**原因：** 匿名函数捕获局部变量时，编译器生成闭包类（引用类型），产生堆分配。

**陷阱：**
- 闭包类的生命周期跟随匿名函数。若将匿名函数存入 List 等容器，资源无法释放
- 闭包类可能很大

**优化：**
- 及时释放引用
- 用 struct 闭包类替代，退出局部作用域时自动释放（struct 在栈上）

### struct 栈分配实验

```csharp
public struct A {
    public Inner inner = new Inner();  // Inner 是 class，string 是引用类型
    int i = 100;
    int i1 = 100;
}
```

- struct 本身在栈上，但其引用类型字段仍在堆上分配
- 实验验证：创建含引用类型字段的 struct 仍然会产生堆内存分配

### UniTask 线程技巧

不需要用到原生 Unity 组件时，可用 `Task.RunOnThreadPool` 放到线程池执行。切回主线程：

```csharp
UniTask.SwitchToMainThread().GetAwaiter().OnCompleted(() => {
    try {
        action();
    } finally {
        mutex.Set();
    }
});
mutex.WaitOne();
```

---

## 七、资源与内存

### material vs sharedMaterial

- `renderer.material`：创建新材质实例，仅影响当前对象，不修改原始材质
- `renderer.sharedMaterial`：直接操作原始材质，影响所有引用该材质的对象

**MaterialPropertyBlock**：替代 `material.SetColor()` 动态修改属性，避免生成大量材质实例、打断合批。

### 实例化材质球问题

`new Material()` 之后必须释放：
1. 直接 `Destroy(material)`
2. `Resources.UnloadUnusedAssets()`
3. 使用 `MaterialPropertyBlock` 替代

### AssetBundle 卸载

- `Unload(false)`：断开索引与资源的连接，不销毁已加载的资源。再次加载会产生冗余
- `Unload(true)`：完全销毁所有资源，可能造成引用丢失。通常在切换场景时使用

### LZ4 vs LZMA

- **LZ4**：按块压缩，压缩率低但读取速度快，支持按需加载
- **LZMA**：流式压缩，压缩率高但只能流式读取

### 图片压缩格式

ERC 和 ASTC — 移动端常用纹理压缩格式。

---

## 八、JSON 与 Protobuf

### JSON (JsonUtility)

```csharp
string json = JsonUtility.ToJson(obj);
MyType obj = JsonUtility.FromJson<MyType>(json);
```

### .NET System.Text.Json

```csharp
string json = JsonSerializer.Serialize(obj);
MyType obj = JsonSerializer.Deserialize<MyType>(json);
```

### Protobuf 序列化

```csharp
// 序列化
MemoryStream ms = new MemoryStream();
msg.WriteTo(ms);
ms.Position = 0;
ms.Read(bytes, 0, len);

// 反序列化
var obj = MessageType.Parser.ParseFrom(bytes);
```

使用反射通用解码：

```csharp
public static bool TryDecodeMsg(byte[] input, out Type type, out object obj) {
    int enum_num = (int)BitConverter.ToInt16(input.AsSpan(0, 2).ToArray());
    type = Type.GetType("Protomsg." + Enum.GetName(typeof(KCP_MessageNum), enum_num));
    if (type == null) return false;
    PropertyInfo pi = type.GetProperty("Parser");
    MessageParser parser = (MessageParser)pi.GetValue(null);
    obj = parser.ParseFrom(input.AsSpan(2, input.Length - 2).ToArray());
    return obj != null;
}
```

### byte[] 拼接工具

```csharp
public static byte[] ConcatBytes(params byte[][] sources) {
    int total = sources.Sum(o => o.Length);
    byte[] res = new byte[total];
    int offset = 0;
    foreach (var src in sources) {
        src.CopyTo(res, offset);
        offset += src.Length;
    }
    return res;
}
```

### PlayerPrefs / JSON / 二进制 比较

- **PlayerPrefs**：键值对存储，轻量级，适合简单配置数据
- **JSON**：字符串格式，适合网络传输和可读性
- **二进制**：体积最小，适合大量数据和网络协议

---

## 九、网络请求 (UnityWebRequest)

```csharp
// 创建请求
UnityWebRequest request = UnityWebRequest.Get(url);
// 或 POST
UnityWebRequest request = UnityWebRequest.Post(url, form);

// 异步发送
var operation = request.SendWebRequest();
yield return operation;

// 获取结果
if (request.result == UnityWebRequest.Result.Success) {
    string text = request.downloadHandler.text;
    byte[] data = request.downloadHandler.data;
} else {
    Debug.LogError(request.error);
}
```

**常用属性：** `result`（枚举）、`error`（string）、`timeout`、`responseCode`、`downloadHandler`/`uploadHandler`、`SendWebRequest`、`SetRequestHeader`、`GetResponseHeader`

---

## 十、编辑器扩展速查

### 核心文件夹

- **Editor/**：编辑器扩展代码，可放任意位置，可有多个，不会打包到游戏
- **Editor Default Resources/**：编辑器资源，通过 `EditorGUIUtility.Load()` 加载
- **Gizmos/**：存放 `Gizmos.DrawIcon()` 的图片资源，需放在 Assets 根目录

### MenuItem

```csharp
// 基本菜单
[MenuItem("MyMenu/Do Something")]
static void DoSomething() { }

// 带验证的菜单（第二个参数 true 表示验证函数）
[MenuItem("MyMenu/Log Selected", true)]
static bool ValidateLogSelected() => Selection.activeTransform != null;

// 快捷键：% = Ctrl/Cmd, # = Shift, & = Alt, _g = 字母 g
[MenuItem("MyMenu/Do Something %g")]  // Ctrl+G
static void DoWithShortcut() { }

// 优先级（相差 >10 分栏）
[MenuItem("GameObject/MyCategory/Custom", false, 10)]
static void CreateCustom(MenuCommand cmd) { }
```

**快捷键符号表：**

| 符号 | 按键 |
|------|------|
| `%`  | Ctrl / Command |
| `#`  | Shift |
| `&`  | Alt |
| `LEFT`/`RIGHT`/`UP`/`DOWN` | 方向键 |
| `F1`-`F12` | F 功能键 |
| `_g` | 字母 g |

### 右键菜单

```csharp
// 组件右键菜单
[MenuItem("CONTEXT/Rigidbody/Double Mass")]
static void DoubleMass(MenuCommand command) {
    Rigidbody body = (Rigidbody)command.context;
    body.mass *= 2;
}

// ContextMenu / ContextMenuItem
[ContextMenu("Do Something")]
void DoSomething() { }

[ContextMenuItem("Reset", "ResetValue")]
[SerializeField] string value = "";
```

### Selection 类

```csharp
Selection.activeGameObject;    // 第一个选中的对象
Selection.gameObjects;          // 选中的多个对象（含预制体）
Selection.objects;              // 选中的多个对象

// 批量操作
foreach (object obj in Selection.objects) {
    DestroyImmediate(obj);
}
```

### 编辑器资源操作

```csharp
// 加载资源
AssetDatabase.LoadAssetAtPath<GameObject>("Assets/Path/Prefab.prefab");
// 保存
AssetDatabase.SaveAssets();
AssetDatabase.Refresh();
// 场景
EditorSceneManager.SaveScene(EditorSceneManager.GetActiveScene());
```

### 代码打开资源

```csharp
AssetDatabase.OpenAsset(target, lineNumber, columnNumber);
```

### Mono.Cecil

一个 .NET 框架，可加载浏览现有程序集并进行动态修改和保存。用于代码结构分析。

### Editor 按钮变色

```csharp
// 方法1：使用 GUI.backgroundColor
Color original = GUI.backgroundColor;
GUI.backgroundColor = Color.green;
if (GUILayout.Button("刷新")) { /* ... */ }
GUI.backgroundColor = original;

// 方法2：覆盖半透明颜色（选中高亮）
if (GUILayout.Button(content, style)) { selected = item; }
if (selected == item) {
    Rect rect = GUILayoutUtility.GetLastRect();
    EditorGUI.DrawRect(rect, new Color(0, 0.15f, 0.62f, 0.3f));
}
```

### 脚本宏定义管理

```csharp
// 获取
PlayerSettings.GetScriptingDefineSymbolsForGroup(BuildTargetGroup.Standalone);
// 设置
PlayerSettings.SetScriptingDefineSymbolsForGroup(BuildTargetGroup.Standalone, "OPEN_LOG;ENABLE_DEBUG");
```

全平台批量操作（Standalone / iOS / Android）。

### 编辑器首选设置

切换外部编辑器：`Edit → Preferences → External Tools` → 选择 Visual Studio / VS Code

### ParrelSync

[GitHub 项目](https://github.com/VeriorPies/ParrelSync) — Unity 多开/项目克隆工具。

---

## 十一、面试精华

### 有限状态机 (FSM)

```csharp
public abstract class FSMState {
    public Dictionary<Transition, StateID> map = new();
    protected StateID stateID;
    public void AddTransition(Transition trans, StateID id) { map.Add(trans, id); }
    public StateID GetOutputState(Transition trans) => map.ContainsKey(trans) ? map[trans] : StateID.NullStateId;
    public virtual void DoBeforeEntering() { }
    public virtual void DoBeforeLeaving() { }
    public virtual void Reason() { }
    public virtual void Act() { }
}

public class FSMSystem {
    private List<FSMState> states;
    private FSMState currentState;
    public void PerformTransition(Transition trans) {
        StateID id = currentState.GetOutputState(trans);
        currentState.DoBeforeLeaving();
        currentState = states.Find(s => s.ID == id);
        currentState.DoBeforeEntering();
    }
}
```

### 行为树基础节点

- **Sequencer（顺序节点）**：按序执行子节点，任一失败则 Sequencer 失败（AND 逻辑）
- **Selector（选择节点）**：全部失败才返回失败（OR 逻辑）

### 三消算法思路

- 策划配置关卡 + 自动化检测是否可过关
- 评分函数：贪心每步最优解（坐标移动分、目标分、留存分等）
- 蒙特卡洛树搜索博弈：考虑当前最优 vs 后续操作空间

### 柏林噪声 vs 纯随机

柏林噪声生成数值具有自然随机性和平滑性，避免纯随机的不连贯，适合地形生成。

### 子弹穿透检测

使用前后帧位置计算路径，进行射线/形状检测，避免高速子弹穿透碰撞体。

### 多相机叠加

- 多相机同时激活会叠加渲染
- 通过 `depth` 控制渲染顺序
- `Depth Only`：先渲染 depth 小的相机作为背景，后渲染 depth 大的相机（Clear Flag = Depth Only 时融合前后相机内容）
- 特定区域可使用 RenderTexture 划定范围

### 事件中心

通过 `Dictionary<Type, Delegate>` 实现事件注册与分发。

### C# List 内部实现

- 默认扩容为 2 的整数倍
- `Remove` / `Insert` 内部使用 `Array.Copy`
- `ToArray()` 复制新数组，有大量内存分配

---

## 参考资源

- [Unity 面试精华 (语雀)](https://www.yuque.com/g/chengxuyuanchangfeng/qxodkp)
- [Unity 编辑器扩展文档](https://docs.unity.cn/cn/2023.2/Manual/ExtendingTheEditor.html)
- [Custom Editors](https://docs.unity.cn/cn/2019.4/Manual/editor-CustomEditors.html)
- [Selection API](https://docs.unity.cn/cn/2023.2/ScriptReference/Selection.html)
- [网易游戏课堂](https://game.academy.163.com/course/careerArticle?course=523)
