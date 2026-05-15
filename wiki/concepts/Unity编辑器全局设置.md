---
title: "Unity 编辑器全局设置"
type: concept
updated: 2026-05-11
tags: [unity, editor, menu-item, selection, context-menu]
---

# Unity 编辑器全局设置

Unity 编辑器扩展的基础全局机制，涵盖文件夹结构、菜单系统和选中对象访问。

## 编辑器文件夹

Unity 通过命名约定识别三类特殊文件夹：

| 文件夹 | 位置要求 | 加载方式 | 打包 |
|--------|---------|---------|------|
| `Editor` | 任意位置、可多个 | 自动编译 | ❌ 排除 |
| `Editor Default Resources` | 根目录、仅一个 | `EditorGUIUtility.Load()` | ❌ 排除 |
| `Gizmos` | 根目录 | `Gizmos.DrawIcon()` | ❌ 排除 |

> Editor 文件夹中脚本仅在编辑器下编译运行，构建时不会打包进游戏。

## MenuItem 菜单系统

向 Unity 顶部菜单栏添加自定义选项。

```csharp
[MenuItem("路径/菜单名", isValidateFunction, priority)]
```

- **priority** 控制排序，差值 > 10 显示分隔线。常用区间：`0-100` 靠前、`1000` 默认、`2000+` 靠后
- **Validate 函数** 配对使用：同名菜单方法，第一个参数 `false` 为执行、`true` 为验证（返回 bool 控制是否可用）
- **快捷键**：`%`=Ctrl、`#`=Shift、`&`=Alt、`_字母`

## 组件右键菜单

### CONTEXT 菜单

为 Inspector 中组件标题栏添加右键菜单，格式 `"CONTEXT/组件类名/菜单名"`。方法接受 `MenuCommand cmd`，通过 `cmd.context` 获取组件引用。

```csharp
[MenuItem("CONTEXT/Rigidbody/Init")]
public static void Init(MenuCommand cmd) {
    Rigidbody rb = cmd.context as Rigidbody;
}
```

> 普通 `[MenuItem]` 中 `MenuCommand` 始终为 null。

### ContextMenu（齿轮菜单）

`[ContextMenu("名称")]` 标记在 public 方法上，为组件右上角齿轮菜单添加选项。

### ContextMenuItem（字段菜单）

`[ContextMenuItem("菜单名", "回调方法名")]` 标记在字段上，为 Inspector 中该字段添加右键菜单。

## Selection 选中管理

`UnityEditor.Selection` 提供对编辑器选中对象的访问：

| 属性/方法 | 说明 |
|-----------|------|
| `activeGameObject` | Inspector 中当前活跃的游戏对象 |
| `activeObject` | 当前活跃对象 |
| `gameObjects` | 场景中选中的 GameObject 数组 |
| `objects` | 未过滤的选中对象数组 |
| `assetGUIDs` | Project 窗口选中资源的 GUID 数组 |
| `selectionChanged` | 选中变化回调事件 |
| `GetFiltered<T>(mode)` | 按类型和模式过滤选中对象 |

```csharp
// 订阅选中变化（需取消订阅避免泄漏）
private void OnEnable()  => Selection.selectionChanged += OnSelectionChanged;
private void OnDisable() => Selection.selectionChanged -= OnSelectionChanged;
```

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles
- [[sources/Unity编辑器全局-摘要|来源摘要]]
