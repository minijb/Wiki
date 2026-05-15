---
title: "Unity 编辑器窗口"
type: concept
updated: 2026-05-11
tags: [unity, editor, window, editor-window]
---

# Unity 编辑器窗口

Unity 提供三种窗口类型，从简单向导到完全自定义工具窗口。

## 窗口类型对比

| | ScriptableWizard | EditorWindow | PopupWindowContent |
|---|---|---|---|
| **复杂度** | 低 | 高 | 低 |
| **持久化** | 一次性任务 | 可保持打开 | 临时 |
| **定位** | 居中弹出 | 自由停靠 | 跟随触发点 |
| **UI** | 自动布局+预定义按钮 | 完全自定义 | 完全自定义 |

## ScriptableWizard

适合导入、设置、创建资源等一次性向导任务。

- **`OnWizardUpdate()`** — 输入变化时调用，设置 `helpString`/`errorString`/`isValid`
- **`OnWizardCreate()`** — 点击确定按钮
- **`OnWizardOtherButton()`** — 点击取消或其他按钮
- **`DrawWizardGUI()`** — 返回 `true` 表示使用自定义绘制

```csharp
ScriptableWizard.DisplayWizard<MyWizard>("Title", "确定", "取消");
```

## EditorWindow

完全自定义的编辑器窗口，可停靠、可保持状态。

```csharp
var window = GetWindow<MyWindow>("My Tool");
window.Show();
```

### 窗口管理

| 方法 | 说明 |
|------|------|
| `GetWindow<T>(title)` | 获取或创建单例窗口 |
| `ShowUtility()` | 显示为浮动工具窗口 |
| `minSize` / `maxSize` | 限制窗口尺寸 |
| `Close()` / `Focus()` | 关闭 / 聚焦 |

### 消息方法

| 方法 | 频率 / 时机 |
|------|------------|
| `OnGUI()` | IMGUI 绘制，约每帧 |
| `Update()` | 约 10fps |
| `OnInspectorUpdate()` | 约 50fps（仅编辑器运行时） |
| `OnEnable()` / `OnDisable()` | 创建/关闭 |
| `OnFocus()` / `OnLostFocus()` | 焦点变化 |
| `OnSelectionChange()` | 选中变化 |
| `OnProjectChange()` | Project 资源变化 |
| `OnHierarchyChange()` | Hierarchy 变化 |

> 新项目推荐用 UI Toolkit（`CreateGUI()` + UXML/USS）替代 `OnGUI()` IMGUI。

## PopupWindowContent

轻量级临时弹窗，点击外部自动关闭。

```csharp
PopupWindow.Show(rect, new MyPopup());
```

## 参见

- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + SerializedProperty
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Gizmos / Handles
- [[sources/Unity编辑器窗口-摘要|来源摘要]]
