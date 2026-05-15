---
title: Unity 自定义Inspector
date: 2026-03-16
tags:
  - unity
  - editor
  - inspector
type: framework
aliases:
  Inspector
description: Unity自定义Inspector面板
status: archived
draft: false
archived-to: raw/gamedev/editor-extensions/Unity自定义Inspector.md
---

## 概述

本页介绍如何通过 `[CustomEditor]` 自定义 MonoBehaviour 的 Inspector 面板，包括两种实现方式（直接访问 Target vs SerializedProperty）、常用布局控件、数组处理和 Scene 视图扩展。

## 目标组件

```csharp
public class CustomEditorTest : MonoBehaviour 
{
    [Space(10)]
    public int     intValue;
    public bool    boolValue;
    public Vector2 v2;
    public float[] floatArray = new float[] {1.0f, 2.0f, 3.0f};
}
```

## 方式 1：直接访问 Target

```csharp
[CanEditMultipleObjects, CustomEditor(typeof(CustomEditorTest))]
public class CustomEditorTestEditor : Editor
{
    private CustomEditorTest _target => target as CustomEditorTest;

    public override void OnInspectorGUI()
    {
        _target.intValue = EditorGUILayout.IntField("IntValue", _target.intValue);
    }
}
```

- 简单直接，代码少
- 不支持自动 Undo — 需要手动 `Undo.RecordObject(_target, "Change")`
- 不支持 Prefab 覆盖、多对象编辑时需额外处理

## 方式 2：SerializedProperty（推荐）

```csharp
[CanEditMultipleObjects, CustomEditor(typeof(CustomEditorTest))]
public class CustomEditorTestEditor : Editor
{
    private SerializedProperty intValueProp;

    private void OnEnable()
    {
        intValueProp = serializedObject.FindProperty("intValue");
    }

    public override void OnInspectorGUI()
    {
        serializedObject.Update();
        EditorGUILayout.PropertyField(intValueProp);
        serializedObject.ApplyModifiedProperties();
    }
}
```

- **自动支持 Undo**、Prefab 覆盖、多对象统一编辑
- 通过 `serializedObject.Update()` / `ApplyModifiedProperties()` 管理序列化状态
- 需要序列化感知（Undo/Prefab/多选）就用 SerializedProperty

## 常用布局

`EditorGUILayout`（自动布局）比 `EditorGUI`（手动 Rect 定位）更便捷，适合大多数 Inspector 场景；`EditorGUI` 适合需要精确像素控制的场景。

```csharp
public override void OnInspectorGUI()
{
    serializedObject.Update();

    // 横向排列
    EditorGUILayout.BeginHorizontal();
    EditorGUILayout.PropertyField(intValueProp);
    if (GUILayout.Button("Reset"))
        intValueProp.intValue = 0;
    EditorGUILayout.EndHorizontal();

    // 可折叠分组
    boolValueProp.isExpanded = EditorGUILayout.Foldout(boolValueProp.isExpanded, "Advanced");
    if (boolValueProp.isExpanded)
    {
        EditorGUI.indentLevel++;
        EditorGUILayout.PropertyField(v2Prop);
        EditorGUI.indentLevel--;
    }

    // 只读区域
    GUI.enabled = false;
    EditorGUILayout.PropertyField(someReadOnlyProp);
    GUI.enabled = true;

    // 帮助/提示信息
    EditorGUILayout.HelpBox("This is a tip", MessageType.Info);

    serializedObject.ApplyModifiedProperties();
}
```

| 布局控件 | 说明 |
|----------|------|
| `EditorGUILayout.BeginHorizontal()` / `EndHorizontal()` | 横向排列 |
| `EditorGUILayout.BeginVertical()` / `EndVertical()` | 纵向排列 |
| `EditorGUI.indentLevel` | 控制缩进层级 |
| `EditorGUILayout.Foldout()` | 可折叠分组 |
| `EditorGUILayout.Space()` | 空白行 |
| `EditorGUILayout.HelpBox()` | 信息/警告/错误提示框 |
| `GUI.enabled` | 禁用/启用控件交互 |
| `EditorGUILayout.Separator()` | 分隔线 |

## 数组处理

```csharp
private SerializedProperty floatArrayProp;

private void OnEnable()
{
    floatArrayProp = serializedObject.FindProperty("floatArray");
}

public override void OnInspectorGUI()
{
    serializedObject.Update();

    // 绘制数组（自动处理增删元素）
    EditorGUILayout.PropertyField(floatArrayProp, true);

    // 手动操作数组
    if (GUILayout.Button("Add Element"))
    {
        floatArrayProp.InsertArrayElementAtIndex(floatArrayProp.arraySize);
    }

    serializedObject.ApplyModifiedProperties();
}
```

| 数组方法 | 说明 |
|----------|------|
| `arraySize` | 数组长度 |
| `GetArrayElementAtIndex(i)` | 获取第 i 个元素的 SerializedProperty |
| `InsertArrayElementAtIndex(i)` | 在位置 i 插入新元素 |
| `DeleteArrayElementAtIndex(i)` | 删除位置 i 的元素 |

## Scene 视图扩展

`CustomEditor` 还可覆写 `OnSceneGUI()` 在 Scene 视图绘制交互元素。

```csharp
[CustomEditor(typeof(CustomEditorTest))]
public class CustomEditorTestEditor : Editor
{
    private void OnSceneGUI()
    {
        var t = target as CustomEditorTest;
        Handles.Label(t.transform.position + Vector3.up * 2, $"Int: {t.intValue}");
    }
}
```

> `OnSceneGUI` 仅在 Editor 脚本中可用。详见 [[concepts/Unity Gizmos 调试|Gizmos 调试]]。

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem、Selection
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles
