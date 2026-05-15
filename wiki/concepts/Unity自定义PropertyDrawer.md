---
title: "Unity 自定义 PropertyDrawer"
type: concept
updated: 2026-05-11
tags: [unity, editor, property-drawer, attribute]
---

# Unity 自定义 PropertyDrawer

通过自定义 `PropertyAttribute` + `PropertyDrawer` 为 Inspector 中任意字段创建自定义绘制逻辑。

## 原理

Unity 编辑器扩展的本质是 **C# 自定义 Attribute + 反射**。定义 Attribute 标记数据，反射读取标记并驱动行为。`[CustomEditor]`、`[CustomPropertyDrawer]` 等都基于此机制。

## 三步实现

### 1. 定义 PropertyAttribute

```csharp
public class ShowTimeAttribute : PropertyAttribute
{
    public readonly bool showHour;
    public ShowTimeAttribute(bool showHour = false) => this.showHour = showHour;
}
```

> `PropertyAttribute.order` 属性控制多个 Drawer 的执行顺序（值越小越先执行）。

### 2. 定义 PropertyDrawer

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
        // 类型校验
        if (property.propertyType != SerializedPropertyType.Integer)
        {
            EditorGUI.HelpBox(position, $"{label.text} must be int", MessageType.Error);
            return;
        }

        var attr = attribute as ShowTimeAttribute;

        // 上半行：整数输入
        var inputRect = new Rect(position.x, position.y, position.width, position.height / 2);
        property.intValue = EditorGUI.IntField(inputRect, label, Mathf.Max(0, property.intValue));

        // 下半行：时间预览
        var labelRect = new Rect(position.x, position.y + position.height / 2, position.width, position.height / 2);
        EditorGUI.LabelField(labelRect, "Time", FormatTime(property.intValue, attr.showHour));
    }

    private static string FormatTime(int seconds, bool showHour)
    {
        int h = seconds / 3600, m = (seconds % 3600) / 60, s = seconds % 60;
        return showHour ? $"{h:D2}:{m:D2}:{s:D2}" : $"{m:D2}:{s:D2}";
    }
}
```

### 3. 使用

```csharp
public class Test : MonoBehaviour
{
    [ShowTime(true)]
    public int time = 3661; // Inspector 显示 "01:01:01"
}
```

## 关键 API

| API | 说明 |
|-----|------|
| `PropertyDrawer.OnGUI(Rect, SerializedProperty, GUIContent)` | 自定义绘制逻辑 |
| `PropertyDrawer.GetPropertyHeight()` | 控制 Inspector 行高 |
| `attribute` | 获取关联的 Attribute 实例 |
| `EditorGUI.IntField/HelpBox/LabelField` | Rect 定位的控件 |

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor 完整组件面板
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem
- [[sources/Unity自定义PropertyDrawer-摘要|来源摘要]]
