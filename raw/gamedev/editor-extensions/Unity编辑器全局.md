---
title: Unity 编辑器全局设置
date: 2026-03-16
tags:
  - unity
  - editor
type: framework
aliases:
  编辑器全局
description: Unity编辑器全局配置 — Editor 文件夹、MenuItem、ContextMenu、Selection
status: archived
draft: false
archived-to: raw/gamedev/editor-extensions/Unity编辑器全局.md
---

## 概述

本页覆盖 Unity 编辑器扩展的基础全局机制：编辑器文件夹结构、`[MenuItem]` 菜单系统、`ContextMenu`/`ContextMenuItem`、`Selection` 类等。适合需要编写编辑器工具脚本的开发者。

## 编辑器相关文件夹

### Editor

- 可以放在任何文件夹下，可以存在多个
- **不会打包**：Unity 通过命名约定自动识别 — 名为 `Editor` 的文件夹中的脚本仅在编辑器编译，构建时排除
- 放编辑器脚本、CustomEditor、EditorWindow、PropertyDrawer 等

### Editor Default Resources

- 必须放在 Assets 根目录下，且只能有一个
- 存放编辑器脚本引用的资源（纹理、字体、USS StyleSheet 等）
- 通过 `EditorGUIUtility.Load()` 加载，资源名不需要后缀
- **不会打包**进游戏
- 与 `Resources` 的区别：`Resources` 中的资源会打包进游戏

```csharp
// Editor Default Resources 加载（不需要后缀名）
Texture2D tex = EditorGUIUtility.Load("MyIcon") as Texture2D;

// Resources 加载（需要放在 Resources 文件夹，打包进游戏）
Texture2D tex = Resources.Load<Texture2D>("MyIcon");
```

### Gizmos

- 必须放在 Assets 根目录下
- 存放 Gizmos 图标资源（通过 `Gizmos.DrawIcon()` 使用）
- **不会打包**进游戏

## MenuItem 菜单项

`public MenuItem (string itemName, bool isValidateFunction, int priority);`

三个参数：
1. 菜单路径
2. 是否是验证函数 — 用于判断当前是否符合执行条件
3. priority 优先级，默认值为 1000

优先级数值差大于 10 时会出现分栏线（分隔线）。常用区间：`0-100` 靠前、`1000` 默认、`2000+` 靠后。

```csharp
// 验证函数必须与同名菜单方法配对，返回 bool
[MenuItem("MyTools/Do Something", false)]
public static void DoSomething() { }

[MenuItem("MyTools/Do Something", true)]
public static bool DoSomethingValidate()
{
    return Selection.activeGameObject != null; // 选中对象时才可用
}
```

### 快捷键

| 符号                 | 按键          |
| ------------------ | ----------- |
| %                  | Ctrl/Command |
| #                  | Shift       |
| &                  | Alt         |
| LEFT/RIGHT/UP/DOWN | 方向键         |
| F1-F2              | F功能键        |
| _g                 | 字母g         |

`[MenuItem("MyTools/test1 %_q")]` — 快捷键为 Ctrl + Q

## CONTEXT 组件右键菜单

为组件 Inspector 标题栏右键菜单添加选项。

`[MenuItem("CONTEXT/组件类型名/菜单名")]`

```csharp
[MenuItem("CONTEXT/Rigidbody/Init")]
public static void InitRigidbody(MenuCommand cmd)
{
    Rigidbody rb = cmd.context as Rigidbody;
    rb.mass = 10f;
}
```

`Rigidbody` 是组件类名（不含命名空间）。若组件在自定义命名空间中，需写完整类名。

## MenuCommand

CONTEXT 菜单方法接受 `MenuCommand` 参数，通过 `cmd.context` 获取当前操作的组件引用。

> 注意：普通 `[MenuItem]`（非 CONTEXT）方法中 `MenuCommand` 为 null。

```csharp
[MenuItem("CONTEXT/PlayerHealth/Init")]
public static void Init(MenuCommand cmd)
{
    PlayerHealth health = cmd.context as PlayerHealth;
}
```

## ContextMenu — 组件齿轮菜单

给 MonoBehaviour 组件右上角的**小齿轮菜单**添加选项，点击后调用当前组件中对应的方法。

- 方法必须是 `public` 或声明了 `ContextMenu` 的类内部可访问
- 方法签名无参数或仅接受 `MenuCommand` 参数

```csharp
public class MyComponent : MonoBehaviour
{
    [ContextMenu("Reset Values")]
    public void ResetValues()
    {
        // 齿轮菜单中会出现 "Reset Values" 选项
    }
}
```

## ContextMenuItem — 字段右键菜单

给 Inspector 中**某个字段**的右键菜单添加选项，点击后调用指定方法。

- 第一个参数：菜单显示名称
- 第二个参数：回调方法名（字符串形式）
- 方法与字段必须在同一个类中

```csharp
public class MyComponent : MonoBehaviour
{
    [ContextMenuItem("Randomize", "RandomizeHealth")]
    public float health;

    private void RandomizeHealth()
    {
        health = Random.Range(0f, 100f);
    }
}
```

## Selection — 访问编辑器选中对象

`UnityEditor.Selection` 提供对编辑器当前选中对象的访问，可用于 Hierarchy、Project 窗口等。

### 常用属性

| 属性 | 返回类型 | 说明 |
|------|----------|------|
| `activeGameObject` | `GameObject` | Inspector 中当前活跃的游戏对象 |
| `activeObject` | `Object` | 当前活跃对象 |
| `activeTransform` | `Transform` | 当前活跃对象的 Transform |
| `gameObjects` | `GameObject[]` | 场景中选中的 GameObject 数组 |
| `objects` | `Object[]` | 未过滤的选中对象数组 |
| `assetGUIDs` | `string[]` | Project 窗口中选中资源的 GUID 数组 |
| `selectionChanged` | `Action` | 选中变化时的回调 |

### 常用方法

| 方法 | 说明 |
|------|------|
| `Contains(Object obj)` | 检查某对象是否在当前选中中 |
| `GetFiltered<T>(SelectionMode mode)` | 按类型和模式过滤选中对象 |
| `GetTransforms(SelectionMode mode)` | 按模式获取 Transform 数组 |

### SelectionMode 枚举

| 值 | 说明 |
|----|------|
| `Unfiltered` | 返回所有对象 |
| `TopLevel` | 仅顶层对象（不含子物体） |
| `Deep` | 包含子物体 |
| `ExcludePrefab` | 排除预制体 |
| `Editable` | 仅可直接修改的对象 |
| `Assets` | 仅资源 |
| `DeepAssets` | 包含子资源 |

```csharp
// 获取选中的第一个 GameObject
GameObject go = Selection.activeGameObject;

// 过滤获取当前选中的 MonoBehaviour
var scripts = Selection.GetFiltered<MonoBehaviour>(SelectionMode.Editable);

// 订阅选中变化（记得取消订阅避免泄漏）
private void OnEnable()  => Selection.selectionChanged += OnSelectionChanged;
private void OnDisable() => Selection.selectionChanged -= OnSelectionChanged;

private void OnSelectionChanged()
{
    Debug.Log($"选中了 {Selection.count} 个对象");
}
```

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — 自定义 PropertyAttribute + PropertyDrawer
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles
