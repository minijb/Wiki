---
title: Unity 编辑器特性
date: 2026-03-16
tags:
  - unity
  - editor
  - attribute
type: framework
aliases:
  编辑器特性
description: Unity编辑器特性Attributes速查表
status: archived
draft: false
archived-to: raw/gamedev/editor-extensions/Unity编辑器特性速查.md
---

## 概述

本页是 Unity 编辑器常用 C# Attribute 的速查表，按作用目标分为属性特性、方法特性、类特性三类。适合快速查阅某个 Attribute 的用途和参数。

## 属性特性

Inspector 中字段的显示与行为控制。

- `[Range(0,100)]` — 限制数值范围，Inspector 中显示为滑块
- `[Min(0)]` / `[Max(100)]` — 轻量级数值限制，仅限制边界无滑块（Unity 2020.2+）
- `[Multiline(3)]` — 字符串多行显示
- `[TextArea(2,4)]` — 文本输入框，最小最大行数，超过最大可滚动
- `[Delayed]` — 输入完成后（回车或失焦）才提交变更，避免每键触发回调
- `[SerializeField]` — 序列化 private 字段，使其显示在 Inspector 中
- `[SerializeReference]` — 多态序列化，支持接口/抽象类字段的序列化（Unity 2019.3+）
- `[NonSerialized]` — 阻止字段序列化。public 字段仍显示在 Inspector 中，但值不会被保存
- `[HideInInspector]` — 在 Inspector 中隐藏 public 字段
- `[FormerlySerializedAs("oldFieldName")]` — 变量重命名后保留原有序列化值不丢失，参数为旧字段名
- `[Header("Value")]` — 字段上方加粗标题
- `[Space(10)]` — 字段上方间距，数值越大间隔越大
- `[Tooltip("xxx")]` — 鼠标悬停提示
- `[ColorUsage(true, true)]` — 显示颜色面板（参数：showAlpha, hdr）
- `[ContextMenu("xx")]` — 组件齿轮菜单添加选项，详见 [[concepts/Unity编辑器全局设置|编辑器全局设置]]
- `[ContextMenuItem("xx", "xxx")]` — 字段右键菜单添加选项，详见 [[concepts/Unity编辑器全局设置|编辑器全局设置]]

可以同时添加多个特性用逗号隔开 `[SerializeField, Range(0,5)]`

## 方法特性

静态方法的执行时机与行为控制。

- `[DrawGizmo]` — 用于 Gizmo 渲染，将调试绘制逻辑与业务代码分离，详见 [[concepts/Unity Gizmos 调试|Gizmos 调试]]
- `[MenuItem]` — 添加菜单项，详见 [[concepts/Unity编辑器全局设置|编辑器全局设置]]
- `[InitializeOnLoadMethod]` — 编辑器启动或脚本重编译后自动执行（静态方法，需在 Editor 文件夹内）
- `[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType)]` — 游戏启动时自动执行（静态方法），可选指定执行时机：
  - `SubsystemRegistration` — 子系统注册阶段（最早）
  - `AfterAssembliesLoaded` — 程序集加载完毕后
  - `BeforeSplashScreen` — 启动画面前
  - `BeforeSceneLoad` — Awake 之前
  - `AfterSceneLoad` — Awake/OnEnable 之后（默认值）
- `[DllImport]` — .NET 原生互操作特性，声明非托管 DLL 的外部方法（非 Unity 专有）

## 类特性

MonoBehaviour / ScriptableObject / Editor 类的行为与元数据控制。

- `[Serializable]` — 序列化一个自定义类，使其字段可显示在 Inspector 中
- `[RequireComponent(typeof(xxx))]` — 挂载时自动添加指定组件，不允许移除
- `[DisallowMultipleComponent]` — 同一对象上不允许挂多个该类或其子类
- `[ExecuteInEditMode]` — 允许脚本在编辑器未运行时执行（旧版，推荐用 `ExecuteAlways`）
- `[ExecuteAlways]` — 在编辑模式和运行时均执行（Unity 2018.3+ 替代 `ExecuteInEditMode`）
- `[CanEditMultipleObjects]` — 允许同时编辑多个选中对象的属性
- `[AddComponentMenu]` — 在 Component 菜单栏内添加组件按钮
- `[CustomEditor]` — 自定义 Inspector，需配合创建 Editor 类，详见 [[concepts/Unity自定义Inspector|自定义 Inspector]]
- `[CustomPropertyDrawer]` — 自定义 PropertyDrawer，详见 [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]]
- `[SelectionBase]` — 在 Scene 视图点击带有此特性的 Prefab 子物体时，自动选中根物体而非子物体
- `[CreateAssetMenu(fileName = "xx", menuName = "Game/xx")]` — 为 ScriptableObject 在 Assets/Create 菜单添加创建选项
- `[HelpURL("https://docs.xxx.com")]` — Inspector 中显示"?"帮助图标，点击打开指定 URL
- `[ImageEffectAllowedInSceneView]` — 允许后处理特效在 Scene 视图中也生效
- `[PreferBinarySerialization]` — ScriptableObject 使用二进制序列化（更快/更小，但不支持文本 diff）
- `[SettingsProvider]` — 为 Project Settings / Preferences 窗口注册自定义设置页（静态方法）
- `[InitializeOnLoad]` — 类级版本，静态构造函数在编辑器启动或重编译后自动执行（需在 Editor 文件夹内）

```csharp
[InitializeOnLoad]
public class MyEditorInitializer
{
    static MyEditorInitializer()
    {
        EditorApplication.update += OnUpdate;
    }

    private static void OnUpdate() { }
}
```

```csharp
[CreateAssetMenu(fileName = "NewConfig", menuName = "Game/Config")]
public class GameConfig : ScriptableObject
{
    public int maxPlayers;
    public float gameSpeed;
}
```

## 参见

- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem、Selection
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles
