---
title: Unity 自定义编辑器特性
date: 2026-03-16
tags:
  - unity
  - editor
  - custom-editor
type: framework
aliases:
  自定义编辑器
description: Unity自定义PropertyAttribute与PropertyDrawer
status: archived
draft: false
archived-to: raw/gamedev/editor-extensions/Unity自定义PropertyDrawer.md
---

## 概述

本页介绍如何通过自定义 Attribute 扩展 Unity 编辑器的 Inspector 显示。第一节讲 C# 自定义 Attribute 与反射（这是 Unity 编辑器扩展的底层机制），第二节讲 Unity 的 `PropertyAttribute` + `PropertyDrawer` 实战。

## C# 自定义 Attribute

`[CustomEditor]`、`[CustomPropertyDrawer]` 等 Unity 编辑器特性正是基于 C# 自定义 Attribute + 反射这一机制。

> `[AttributeUsage]` 可限制 Attribute 的作用目标（类、字段、方法等），实战中建议始终声明。

```csharp
// 1. 定义自定义 Attribute
[AttributeUsage(AttributeTargets.Field)]
public class HttpApiKey : Attribute
{
    public string apiName;
    public HttpApiKey(string name) => apiName = name;
}

// 2. 使用
public class HttpId
{
    [HttpApiKey("Register")]
    public const int registerId = 10001;
    [HttpApiKey("Login")]
    public const int loginId = 10002;
}

// 3. 反射读取
public static void ReadAttributes()
{
    foreach (var field in typeof(HttpId).GetFields())
    {
        var attr = field.GetCustomAttribute<HttpApiKey>();
        if (attr != null)
        {
            int id = (int)field.GetValue(null);
            Debug.Log($"{attr.apiName}: {id}");
        }
    }
}
```

## Unity PropertyAttribute

### 定义 PropertyAttribute

```csharp
public class ShowTimeAttribute : PropertyAttribute
{
    public readonly bool showHour;
    public ShowTimeAttribute(bool showHour = false) => this.showHour = showHour;
}
```

> `PropertyAttribute` 有 `order` 属性，当同一字段有多个 Drawer 时控制执行顺序（值越小越先执行）。

### 定义 PropertyDrawer

```csharp
[CustomPropertyDrawer(typeof(ShowTimeAttribute))]
public class TimerDrawer : PropertyDrawer
{
    public override float GetPropertyHeight(SerializedProperty property, GUIContent label)
    {
        return EditorGUI.GetPropertyHeight(property) * 2;
    }

    public override void OnGUI(Rect position, SerializedProperty property, GUIContent label)
    {
        if (property.propertyType != SerializedPropertyType.Integer)
        {
            EditorGUI.HelpBox(position, $"{label.text} must be int", MessageType.Error);
            return;
        }

        var attr = attribute as ShowTimeAttribute;

        // 上半行：整数输入
        var inputRect = new Rect(position.x, position.y, position.width, position.height / 2);
        property.intValue = EditorGUI.IntField(inputRect, label, Mathf.Max(0, property.intValue));

        // 下半行：显示转换后的时间
        var labelRect = new Rect(position.x, position.y + position.height / 2, position.width, position.height / 2);
        EditorGUI.LabelField(labelRect, "Time", FormatTime(property.intValue, attr.showHour));
    }

    private static string FormatTime(int seconds, bool showHour)
    {
        int h = seconds / 3600;
        int m = (seconds % 3600) / 60;
        int s = seconds % 60;
        return showHour ? $"{h:D2}:{m:D2}:{s:D2}" : $"{m:D2}:{s:D2}";
    }
}
```

### 使用

```csharp
public class Test : MonoBehaviour
{
    [ShowTime(true)]
    public int time = 3661; // Inspector 输入秒数，下方显示 "01:01:01"
}
```

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem、Selection
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles

---

## 实战：Autohook 自动绑定 PropertyDrawer

Autohook 是一个实用的 PropertyDrawer 模式，通过命名约定自动绑定组件引用，消除手动拖拽：

```csharp
// 定义标记性 Attribute
public class AutohookAttribute : PropertyAttribute { }

// 定义 PropertyDrawer
[CustomPropertyDrawer(typeof(AutohookAttribute))]
public class AutohookPropertyDrawer : PropertyDrawer
{
#if UNITY_EDITOR
    public override void OnGUI(Rect position, SerializedProperty property, GUIContent label)
    {
        // 自动查找并绑定组件
        var component = FindAutohookTarget(property);
        if (component != null)
        {
            property.objectReferenceValue = component;
        }
        EditorGUI.PropertyField(position, property, label);
    }
#endif
    private Component FindAutohookTarget(SerializedProperty property)
    {
        var root = property.serializedObject;
        if (root.targetObject is Component)
        {
            var type = GetTypeFromProperty(property);
            var component = (Component)root.targetObject;
            // 在子物体中查找名称匹配 "#字段名" 的 GameObject
            var components = component.GetComponentsInChildren(type, true);
            foreach (var item in components)
            {
                if (item.gameObject.name == "#" + property.name)
                {
                    return item.gameObject.GetComponent(type);
                }
            }
        }
        return null;
    }
    private static System.Type GetTypeFromProperty(SerializedProperty property)
    {
        var parentComponentType = property.serializedObject.targetObject.GetType();
        var fieldInfo = parentComponentType.GetField(property.propertyPath);
        return fieldInfo.FieldType;
    }
}

// 使用
public class PlayerUI : MonoBehaviour
{
    [Autohook] public Button btnStart;     // 自动绑定子物体 "#btnStart"
    [Autohook] public Text txtScore;       // 自动绑定子物体 "#txtScore"
}
```

> [!tip] Autohook 命名约定
> 字段名 `btnStart` 自动查找子物体中名为 `#btnStart` 的 GameObject，并获取对应类型的 Component。`#` 前缀用于区分子物体和普通物体名。

> [!warning] 使用限制
> 1. 子物体名必须唯一（不能有重名的 `#btnStart`）
> 2. 仅适用于 `Component` 类型字段
> 3. `#if UNITY_EDITOR` 确保仅在编辑器中执行查找，运行时零开销
