---
title: Unity 自定义窗口
date: 2026-03-16
tags:
  - unity
  - editor
  - window
type: framework
aliases:
  自定义窗口
description: Unity自定义EditorWindow窗口
status: archived
draft: false
archived-to: raw/gamedev/editor-extensions/Unity编辑器窗口.md
---
jixu 
## 概述

本页介绍 Unity 编辑器中的三种窗口类型：`ScriptableWizard`（向导）、`EditorWindow`（完全自定义）和 `PopupWindowContent`（轻量弹窗），覆盖创建方式、生命周期和适用场景。

## 窗口类型对比

| | ScriptableWizard | EditorWindow | PopupWindowContent |
|------|------|------|------|
| **复杂度** | 低 | 高 | 低 |
| **持久化** | 一次性 | 可保持打开 | 临时 |
| **定位** | 居中弹出 | 自由停靠 | 跟随触发点 |
| **UI** | 自动布局 + 预定义按钮 | 完全自定义 | 完全自定义 |

## ScriptableWizard（向导窗口）

单一目的的工具向导，用于执行特定的一次性任务。

- 内置标准化 UI 布局
- 预定义 "Create"、"Apply"、"Cancel" 等标准按钮
- `helpString` — 帮助信息，始终显示
- `errorString` — 错误信息，仅当 `isValid = false` 时显示
- `isValid` — 设为 false 禁用确定按钮

```csharp
// 启动向导
ScriptableWizard.DisplayWizard<MyWizard>("Title", "确定", "取消");

public class MyWizard : ScriptableWizard
{
    public int someValue;

    // 输入变化时更新预览
    private void OnWizardUpdate()
    {
        helpString = $"当前值: {someValue}";
        if (someValue <= 0)
        {
            errorString = "值必须大于 0";
            isValid = false;
        }
        else
        {
            errorString = "";
            isValid = true;
        }
    }

    // 点击"确定"
    private void OnWizardCreate()
    {
        // 执行创建逻辑
    }

    // 点击"取消"或其他按钮
    private void OnWizardOtherButton()
    {
        Close();
    }

    // 自定义绘制（返回 true 表示使用了自定义绘制，跳过默认布局）
    protected override bool DrawWizardGUI()
    {
        return base.DrawWizardGUI();
    }
}
```

## EditorWindow（自定义编辑器窗口）

完全自定义的编辑器窗口，可创建任意复杂的工具界面。

- 持久化：窗口保持打开，状态可保存
- 灵活 UI：使用 IMGUI（`OnGUI`）或 UI Toolkit（`CreateGUI` + UXML/USS）
- 可停靠到编辑器布局

```csharp
public class MyWindow : EditorWindow
{
    private bool toggleValue;

    [MenuItem("Tools/My Window")]
    public static void ShowWindow()
    {
        var window = GetWindow<MyWindow>("My Tool");
        window.Show();
    }

    private void OnEnable()
    {
        // 窗口创建或重编译后初始化
        Debug.Log("Window enabled");
    }

    private void OnDisable()
    {
        // 窗口关闭时清理
        Debug.Log("Window disabled");
    }

    private void OnGUI()
    {
        GUILayout.Label("Hello from EditorWindow", EditorStyles.boldLabel);
        toggleValue = EditorGUILayout.Toggle("Toggle", toggleValue);
    }
}
```

### 窗口管理

| 常用方法 | 说明 |
|----------|------|
| `GetWindow<T>(title)` | 获取或创建窗口（单例） |
| `CreateInstance<T>()` | 创建窗口实例（不自动显示） |
| `Show()` | 显示窗口 |
| `ShowUtility()` | 显示为工具窗口（浮动） |
| `Focus()` | 聚焦窗口 |
| `Close()` | 关闭窗口 |
| `minSize` / `maxSize` | 限制窗口最小/最大尺寸 |

### 常用消息方法

| 方法 | 调用频率/时机 |
|------|------|
| `OnEnable()` | 窗口创建或脚本重编译后 |
| `OnDisable()` | 窗口关闭时清理 |
| `OnGUI()` | IMGUI 绘制，约每帧 |
| `Update()` | 约 10fps（编辑器未运行时也调用） |
| `OnInspectorUpdate()` | 约 50fps（仅编辑器运行时） |
| `OnFocus()` | 窗口获得焦点时 |
| `OnLostFocus()` | 窗口失去焦点时 |
| `OnSelectionChange()` | 选中对象变化时 |
| `OnProjectChange()` | Project 资源变化时 |
| `OnHierarchyChange()` | Hierarchy 变化时 |

> UI Toolkit（`CreateGUI` + UXML/USS）是 Unity 推荐的现代编辑器 UI 方案，替代 `OnGUI` IMGUI。IMGUI 适合快速原型和简单窗口。

## PopupWindowContent（弹出窗口）

轻量级临时窗口，点击外部自动关闭。

```csharp
public class MyPopup : PopupWindowContent
{
    public override Vector2 GetWindowSize() => new Vector2(200, 100);

    public override void OnGUI(Rect rect)
    {
        EditorGUILayout.LabelField("Hello from Popup", EditorStyles.boldLabel);
    }
}

// 在 OnGUI 中触发
var popup = new MyPopup();
var rect = new Rect(Event.current.mousePosition, Vector2.zero);
PopupWindow.Show(rect, popup);
```

## 参见

- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem、Selection
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / DrawGizmo / Handles
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
