---
title: Unity 编辑器全局设置
date: 2026-03-16
tags:
  - unity
  - editor
type: framework
aliases:
  编辑器全局
description: Unity编辑器全局配置
draft: false
---


## 1. 编辑器相关的文件夹

**Editor**

- 可以放在任何文件夹下，可以存在多个
- **不会打包**：Unity 构建时自动排除名为 `Editor` 的文件夹（通过 `Assembly Definition` 中 `Editor` 平台标记实现）
- 放编辑器脚本、CustomEditor、EditorWindow、PropertyDrawer 等
- Editor 文件夹中的代码仅在编辑器中编译运行

**Editor Default Resources**

- 必须放在 Assets 根目录下，且只能有一个
- 存放编辑器脚本引用的资源（纹理、字体、USS StyleSheet 等）
- 通过 `EditorGUIUtility.Load()` 加载，资源名不需要后缀
- **不会打包**进游戏
- 与 `Resources` 的区别：`Resources` 中的资源会打包进游戏

**Gizmos**

- 必须放在 Assets 根目录下
- 存放 Gizmos 图标资源（通过 `Gizmos.DrawIcon()` 使用）
- **不会打包**进游戏

## 2. `[MenuItem]` 添加菜单栏按钮

`public MenuItem (string itemName, bool isValidateFunction, int priority);`

三个参数
1. 菜单路径
2. 是否是有效函数 --- 有效函数(判断当前是否符合使用条件)
3. priority 优先级，默认值为1000， 数值差大于10，会出现分栏

快捷键

| 符号                 | 按键          |
| ------------------ | ----------- |
| %                  | Ctr/Command |
| #                  | Shift       |
| &                  | Alt         |
| LEFT/RIGHT/UP/DOWN | 方向键         |
| F1-F2              | F功能键        |
| _g                 | 字母g         |

`[MenuItem("MyTools/test1 %_q")]` --- 快捷键为 Ctrl + Q

## 3. CONTEXT

**组件添加右键菜单选项** <font color="#0070c0">算是比较常用</font>

`[MenuItem("CONTEXT/Rigidbody/Init")]` --- 为 Rigidbody 添加右键菜单Init选项

## 4. MenuCommand

**获取当前操作组件的上下文**

```c#
[MenuItem("CONTEXT/PlayerHealth/Init")]
public static Init(MenuCommand cmd)
{
	// 这里获得了 对应的上下文对象
	PlayerHealth health = cmd.context as PlayerHealth;
}
```

## 5. `[ContextMenu]` — 组件齿轮菜单

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

## 6. `[ContextMenuItem]` — 字段右键菜单

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

## 7. Selection — 访问编辑器选中对象

`UnityEditor.Selection` 提供对编辑器当前选中对象的访问，可用于 Hierarchy、Project 窗口等。

### 静态属性

| 属性 | 返回类型 | 说明 |
|------|----------|------|
| `activeGameObject` | `GameObject` | 当前活跃的游戏对象（Inspector 中显示的） |
| `activeObject` | `Object` | 当前活跃的对象，包含预制体等不可修改对象 |
| `activeTransform` | `Transform` | 当前活跃对象的 Transform（Inspector 中显示的） |
| `activeInstanceID` | `int` | 当前活跃对象的 InstanceID |
| `activeContext` | `Object` | 通过 `SetActiveObjectWithContext` 设置的上下文对象 |
| `gameObjects` | `GameObject[]` | 场景中选中的 GameObject 数组，包含预制体 |
| `objects` | `Object[]` | 未过滤的选中对象数组 |
| `transforms` | `Transform[]` | 顶层选中的 Transform，**不包含**预制体 |
| `instanceIDs` | `int[]` | 未过滤的选中对象 InstanceID 数组 |
| `assetGUIDs` | `string[]` | Project 窗口中选中资源的 GUID 数组 |
| `count` | `int` | 选中对象的数量 |
| `selectionChanged` | `Action` | 选中变化时的回调委托 |

### 静态方法

| 方法 | 说明 |
|------|------|
| `Contains(Object obj)` | 检查某对象是否在当前选中中 |
| `Contains(int instanceID)` | 通过 InstanceID 检查是否被选中 |
| `GetFiltered<T>(SelectionMode mode)` | 按类型和模式过滤选中对象 |
| `GetTransforms(SelectionMode mode)` | 按模式获取 Transform 数组 |
| `SetActiveObjectWithContext(Object obj, Object context)` | 选中对象并设置上下文 |

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

// 订阅选中变化
Selection.selectionChanged += OnSelectionChanged;
```
