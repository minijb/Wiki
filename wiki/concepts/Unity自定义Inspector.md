---
title: "Unity 自定义 Inspector"
type: concept
updated: 2026-05-11
tags: [unity, editor, inspector, custom-editor]
---

# Unity 自定义 Inspector

通过 `[CustomEditor]` 创建 Editor 类，完全自定义 MonoBehaviour 的 Inspector 面板。

## 两种实现方式

### 方式 1：直接访问 Target（简单，有局限）

```csharp
[CanEditMultipleObjects, CustomEditor(typeof(MyComponent))]
public class MyComponentEditor : Editor
{
    private MyComponent _target => target as MyComponent;

    public override void OnInspectorGUI()
    {
        _target.intValue = EditorGUILayout.IntField("IntValue", _target.intValue);
    }
}
```

> 不支持自动 Undo，需手动调用 `Undo.RecordObject(_target, "Change")`。

### 方式 2：SerializedProperty（推荐）

```csharp
[CanEditMultipleObjects, CustomEditor(typeof(MyComponent))]
public class MyComponentEditor : Editor
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

- 自动支持 Undo、Prefab 覆盖、多对象统一编辑
- `serializedObject.Update()` / `ApplyModifiedProperties()` 管理序列化状态

## 常用布局

| 控件 | 说明 |
|------|------|
| `EditorGUILayout.BeginHorizontal()` | 横向排列 |
| `EditorGUILayout.BeginVertical()` | 纵向排列 |
| `EditorGUILayout.Foldout()` | 可折叠分组 |
| `EditorGUILayout.HelpBox()` | 信息/警告/错误提示框 |
| `EditorGUILayout.Space()` / `Separator()` | 空白行 / 分隔线 |
| `GUI.enabled = false` | 禁用控件交互 |
| `EditorGUI.indentLevel++` | 增加缩进 |

> `EditorGUILayout`（自动布局）优先于 `EditorGUI`（手动 Rect 定位），适合大多数 Inspector 场景。

## 数组处理

```csharp
// 自动绘制（可增删元素）
EditorGUILayout.PropertyField(floatArrayProp, true);

// 手动操作
floatArrayProp.InsertArrayElementAtIndex(floatArrayProp.arraySize);
floatArrayProp.DeleteArrayElementAtIndex(index);
SerializedProperty elem = floatArrayProp.GetArrayElementAtIndex(i);
```

## OnSceneGUI — Scene 视图扩展

CustomEditor 还可覆写 `OnSceneGUI()` 在 Scene 视图绘制交互元素。

```csharp
private void OnSceneGUI()
{
    var t = target as MyComponent;
    Handles.Label(t.transform.position + Vector3.up * 2, $"Value: {t.intValue}");
}
```

## 参见

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / Handles
- [[sources/Unity自定义Inspector-摘要|来源摘要]]
