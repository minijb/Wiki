---
title: "Unity 编辑器特性"
type: concept
updated: 2026-05-11
tags: [unity, editor, attribute]
---

# Unity 编辑器特性

Unity 编辑器常用 C# Attribute 速查表，按作用目标分为属性特性、方法特性、类特性三类。

## 属性特性

Inspector 中字段的显示与行为控制。

| Attribute                        | 说明                            |
| -------------------------------- | ----------------------------- |
| `[Range(min, max)]`              | 滑块限制数值范围                      |
| `[Min(0)]` / `[Max(100)]`        | 轻量边界限制（Unity 2020.2+）         |
| `[SerializeField]`               | 序列化 private 字段                |
| `[SerializeReference]`           | 多态序列化，支持接口/抽象类（Unity 2019.3+） |
| `[NonSerialized]`                | 阻止序列化（public 字段仍显示但不保存）       |
| `[HideInInspector]`              | 隐藏 public 字段                  |
| `[FormerlySerializedAs("旧名")]`   | 重命名后保留序列化值                    |
| `[Header("标题")]`                 | 字段上方加粗标题                      |
| `[Space(n)]`                     | 字段上方间距                        |
| `[Tooltip("说明")]`                | 鼠标悬停提示                        |
| `[ColorUsage(showAlpha, hdr)]`   | 颜色面板                          |
| `[TextArea(min, max)]`           | 多行文本输入框                       |
| `[Multiline(n)]`                 | 字符串多行显示                       |
| `[Delayed]`                      | 回车或失焦后才提交变更                   |
| `[ContextMenu("名称")]`            | 齿轮菜单选项                        |
| `[ContextMenuItem("名称", "方法名")]` | 字段右键菜单                        |

> 多个特性可逗号并列：`[SerializeField, Range(0, 5)]`

## 方法特性

静态方法的执行时机与行为。

| Attribute | 说明 |
|-----------|------|
| `[MenuItem]` | 添加菜单项 |
| `[DrawGizmo]` | Gizmo 渲染（Editor 文件夹） |
| `[InitializeOnLoadMethod]` | 编辑器启动 / 脚本重编译后执行 |
| `[RuntimeInitializeOnLoadMethod]` | 游戏启动时执行，5 种时机可选 |
| `[DllImport]` | .NET 原生互操作（非 Unity 专有） |

## 类特性

MonoBehaviour / ScriptableObject / Editor 类的行为与元数据。

| Attribute | 说明 |
|-----------|------|
| `[Serializable]` | 自定义类序列化 |
| `[RequireComponent(typeof(X))]` | 自动添加组件，不允许移除 |
| `[DisallowMultipleComponent]` | 禁止同对象挂多个 |
| `[ExecuteAlways]` | 编辑模式 + 运行时均执行（替代 `ExecuteInEditMode`） |
| `[CanEditMultipleObjects]` | 多对象同时编辑 |
| `[SelectionBase]` | 点击子物体时自动选中根物体 |
| `[CreateAssetMenu]` | ScriptableObject 创建菜单 |
| `[CustomEditor]` | 自定义 Inspector |
| `[CustomPropertyDrawer]` | 自定义 PropertyDrawer |
| `[InitializeOnLoad]` | 类级自动初始化（静态构造函数） |
| `[HelpURL]` | Inspector 帮助图标链接 |
| `[SettingsProvider]` | Project Settings 注册页 |
| `[ImageEffectAllowedInSceneView]` | Scene 视图后处理 |
| `[PreferBinarySerialization]` | 二进制序列化（更快/更小） |
| `[AddComponentMenu]` | Component 菜单按钮 |

## 参见

- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem、Selection
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles
- [[sources/Unity编辑器特性速查-摘要|来源摘要]]
