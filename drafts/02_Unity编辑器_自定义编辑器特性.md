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
description: Unity自定义编辑器CustomEditor
draft: false
---


## 1. C# 自定义 Attribute

```csharp
// 定义自定义 Attribute
public class HttpApiKey : Attribute
{
    public string apiName;
    public HttpApiKey(string name) => apiName = name;
}

// 使用
public class HttpId
{
    [HttpApiKey("Register")]
    public const int registerId = 10001;
    [HttpApiKey("Login")]
    public const int loginId = 10002;
}

// 反射读取
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

> C# 自定义 Attribute + 反射是 Unity 编辑器扩展的基础，`[CustomEditor]`、`[CustomPropertyDrawer]` 等都依赖此机制。


## 2. Unity PropertyAttribute

```csharp
// 1. 定义自定义 Attribute
public class ShowTimeAttribute : PropertyAttribute
{
    public readonly bool showHour;
    public ShowTimeAttribute(bool showHour = false) => this.showHour = showHour;
}

// 2. 定义对应的 PropertyDrawer
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

// 3. 使用
public class Test : MonoBehaviour
{
    [ShowTime(true)]
    public int time = 3661; // Inspector 输入秒数，下方显示 "01:01:01"
}
```