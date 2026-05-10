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
draft: false
---


# Unity编辑器 自定义Inspector面板

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
- **缺点**：不支持 Undo（Ctrl+Z 无效）、不支持 Prefab 覆盖、多对象编辑时需额外处理

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
