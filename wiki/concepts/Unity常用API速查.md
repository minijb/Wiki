---
title: "Unity 常用 API 速查"
type: concept
updated: 2026-06-02
tags: [unity, api, cheatsheet, editor, ui, transform]
---

# Unity 常用 API 速查

涵盖 Unity 日常开发中最常用的 API：Transform、Time、Input、UI、序列化、编辑器扩展和常见模式。

## Transform 与 GameObject

`Transform` 和 `GameObject` 是 Unity 场景操作的基础。两者可以互相获取（`transform.gameObject` / `gameObject.transform`），但不能相互转换。

- `Transform` 不能 `new`、不能 `Destroy`；`GameObject` 可以
- 位移：`transform.Translate()` 可指定 local/world/自身坐标系
- `Vector3.forward` 恒为 `(0, 0, 1)`，而 `transform.forward` 是自身 Z 轴在世界坐标的方向（长度 1）
- 父子关系：`SetParent()`、`Find()`、`GetChild()`、`SetSiblingIndex()`

## Time 系统

| 属性 | 受 timeScale 影响 | 说明 |
|------|:---:|------|
| `deltaTime` | ✅ | 帧间隔 |
| `unscaledDeltaTime` | ❌ | 不受缩放的帧间隔 |
| `fixedDeltaTime` | ✅ | 物理帧间隔 |
| `realtimeSinceStartup` | ❌ | 真实时间 |

- `timeScale = 0` 时 `FixedUpdate` 停止，`Update`/`LateUpdate` 照常
- `Coroutine` 中 `WaitForSeconds` 会卡住，需用 `WaitForSecondsRealtime`

## Input System

**旧版 Input：** `GetKey`（按住）、`GetKeyDown`（按下第一帧）、`GetKeyUp`（松开第一帧）。

**新版 Input System：** 通过 `InputActionAsset` 配置，分离逻辑与设备。
- Action Type：`Value`（连续值）、`Button`（三态）、`Pass Through`（双向）
- 支持事件订阅和直接读取 `ReadValue<T>()`

## UI 系统

- **Canvas Scaler**：`Scale With Screen Size` 按参考分辨率缩放；`Expand`/`Shrink` 处理非等比缩放
- **Content Size Fitter**：自动根据子元素调整尺寸，配合 `LayoutGroup` 使用
- **锚点/Pivot**：锚点控制子物体相对父物体的定位和适配；Pivot 是旋转和缩放的中心
- **点击检测**：Camera 模式用 `ScreenPointToLocalPointInRectangle`；Overlay 模式用 `GetWorldCorners` 转世界坐标
- **OverDraw**：半透明 UI 和粒子是主犯，可通过 Cull Transparent Mesh 和 RectMask2D 优化

## 序列化

- `[SerializeField]` 暴露私有字段给 Inspector
- `[Serializable]` 标记自定义类可序列化
- **ScriptableObject**：可复用数据资源，通过 `CreateAssetMenu` 创建，避免大量数据复制
- **SerializedProperty**：编辑器中使用，表示序列化字段的引用关系

## 常用模式

- **单例**：饿汉式（Awake 中初始化）用于 MonoBehaviour；懒汉式用于普通 C# 类
- **闭包 GC**：匿名函数捕获局部变量生成闭包类（堆分配），及时释放引用或用 struct 替代
- **material vs sharedMaterial**：前者创建新实例（打断合批），后者操作原始材质；推荐用 `MaterialPropertyBlock` 动态修改属性

## JSON 与 Protobuf

- `JsonUtility.ToJson/FromJson` — Unity 内置 JSON
- `System.Text.Json` — .NET 标准库
- Protobuf：二进制序列化，性能优于 JSON，使用 `Parser.ParseFrom` 反序列化

## 编辑器扩展

```csharp
[MenuItem("Tools/MyTool %g")]  // Ctrl+G 快捷键
static void MyTool() { }
```

- **Editor 文件夹**：编辑器代码，不打入游戏包
- **Gizmos 文件夹**：Gizmos.DrawIcon 图片资源
- `Selection.activeGameObject` / `Selection.gameObjects` 获取选中对象
- `AssetDatabase` 操作资源；`ContextMenu`/`ContextMenuItem` 添加右键菜单
- `ScriptingDefineSymbols` 管理编译宏

## 资源管理

- `AssetBundle.Unload(false)` — 断开连接不销毁；`Unload(true)` — 完全销毁
- **LZ4**：块压缩，读取快；**LZMA**：流压缩，压缩率高

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性速查]] — 内置 Attribute 速查
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — 属性绘制
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow/PopupWindowContent
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — 文件夹结构和 MenuItem
- [[concepts/CSharp值类型性能|C# 值类型性能]] — struct/class 分配差异
- [[concepts/CSharp集合框架|C# 集合框架]] — List 内部实现
- [[sources/Unity常用API速查表-摘要|来源摘要]]
