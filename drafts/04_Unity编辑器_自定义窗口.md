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
draft: false
---


## 1. 窗口类型

### 1.1 ScriptableWizard（向导窗口）

- 单一目的的工具：用于执行特定的、一次性任务
- 简化的工作流程：用于导入、设置、创建资源的向导
- 自动布局：内置标准化 UI 布局结构
- 预定义按钮：内置 "Create"、"Apply"、"Cancel" 等标准按钮

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
        isValid = someValue > 0;
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

    // 自定义绘制
    protected override bool DrawWizardGUI()
    {
        return base.DrawWizardGUI();
    }
}
```

### 1.2 EditorWindow（自定义编辑器窗口）

- 完全自定义：可创建任意复杂度的编辑器界面
- 持久化状态：窗口可保持打开，状态可保存
- 灵活的 UI 布局：使用 IMGUI 或 UI Toolkit 自由绘制
- 多标签页支持：可创建复杂的多页面工具

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

| 常用方法 | 说明 |
|----------|------|
| `GetWindow<T>(title)` | 获取或创建窗口（单例） |
| `CreateWindow<T>()` | 始终创建新窗口 |
| `Show()` | 显示窗口 |
| `ShowUtility()` | 显示为工具窗口（浮动） |
| `Focus()` | 聚焦窗口 |
| `Close()` | 关闭窗口 |

### 1.3 PopupWindowContent（弹出窗口）

- 轻量级临时窗口：用于显示临时的、非模态的 UI 元素
- 自动定位：通常显示在触发它的控件附近
- 自动关闭：点击窗口外部时自动关闭
- 无边框设计：简洁的弹出式设计

```csharp
var popup = new MyPopupContent();
var rect = new Rect(Event.current.mousePosition, Vector2.zero);
PopupWindow.Show(rect, popup);
```
