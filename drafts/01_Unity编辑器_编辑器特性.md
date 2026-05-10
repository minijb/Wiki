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
description: Unity编辑器特性Attributes
draft: false
---


## 1. 属性特性

- `[Range(0,100)]` — 限制数值范围，Inspector 中显示为滑块
- `[Min(0)]` / `[Max(100)]` — 轻量级数值限制，仅限制边界无滑块（Unity 2020.2+）
- `[Multiline(3)]` — 字符串多行显示
- `[TextArea(2,4)]` — 文本输入框，最小最大行数，超过最大可滚动
- `[Delayed]` — 输入完成后（回车或失焦）才提交变更，避免每键触发回调
- `[SerializeField]` — 序列化 private 字段，使其显示在 Inspector 中
- `[NonSerialized]` — 不序列化（注意：不隐藏 Inspector）；需配合 `[HideInInspector]` 隐藏
- `[HideInInspector]` — 在 Inspector 中隐藏 public 字段
- `[FormerlySerializedAs("Value")]` — 变量重命名后保留原有序列化值不丢失
- `[ContextMenu("xx")]` — 组件齿轮菜单添加选项
- `[ContextMenuItem("xx", "xxx")]` — 字段右键菜单添加选项
- `[Header("Value")]` — 字段上方加粗标题
- `[Space(10)]` — 字段上方间距，数值越大间隔越大
- `[Tooltip("xxx")]` — 鼠标悬停提示
- `[ColorUsage(true)]` — 显示颜色面板（参数：是否显示 Alpha 通道）

## 2. 方法特性

- `[DrawGizmo]` — 用于 Gizmo 渲染，将调试绘制逻辑与业务代码分离
- `[MenuItem]` — 添加菜单项
- `[InitializeOnLoadMethod]` — 编辑器启动或脚本重编译后自动执行（静态方法，需在 Editor 文件夹内）
- `[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType)]` — 游戏启动时自动执行（静态方法），可选指定执行时机：
  - `SubsystemRegistration` — 子系统注册阶段（最早）
  - `AfterAssembliesLoaded` — 程序集加载完毕后
  - `BeforeSplashScreen` — 启动画面前
  - `BeforeSceneLoad` — Awake 之前
  - `AfterSceneLoad` — Awake/OnEnable 之后（默认值）
- `[DllImport]` — 声明非托管 DLL 的外部方法，用于调用第三方原生插件

## 3. 常用的类的特性

- `[Serializable]` — 序列化一个类，使其可显示在 Inspector 中
- `[RequireComponent(typeof(xxx))]` — 挂载时自动添加指定组件，不允许移除
- `[DisallowMultipleComponent]` — 同一对象上不允许挂多个该类或其子类
- `[ExecuteInEditMode]` — 允许脚本在编辑器未运行时执行（旧版，推荐用 `ExecuteAlways`）
- `[ExecuteAlways]` — 在编辑模式和运行时均执行（Unity 2018.3+ 替代 `ExecuteInEditMode`）
- `[CanEditMultipleObjects]` — 允许同时编辑多个选中对象的属性
- `[AddComponentMenu]` — 在 Component 菜单栏内添加组件按钮
- `[CustomEditor]` — 要自定义 Inspector 需添加此特性
- `[CustomPropertyDrawer]` — 用于自定义 PropertyDrawer
- `[SelectionBase]` — 在 Scene 视图点击带有此特性的 Prefab 子物体时，自动选中根物体而非子物体
- `[CreateAssetMenu(fileName = "xx", menuName = "Game/xx")]` — 为 ScriptableObject 在 Assets/Create 菜单添加创建选项
- `[HelpURL("https://docs.xxx.com")]` — Inspector 中显示"?"帮助图标，点击打开指定 URL
- `[ImageEffectAllowedInSceneView]` — 允许后处理特效在 Scene 视图中也生效
- `[PreferBinarySerialization]` — ScriptableObject 使用二进制序列化（更快/更小，但不支持文本 diff）
- `[SettingsProvider]` — 为 Project Settings / Preferences 窗口注册自定义设置页（静态方法）

可以同时添加多个特性用逗号隔开 `[SerializeField, Range(0,5)]`

