---
title: "Unity 编辑器窗口 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/editor-extensions/Unity编辑器窗口.md"
tags: [unity, editor, window, editor-window]
---

# Unity 编辑器窗口

## 来源

`raw/gamedev/editor-extensions/Unity编辑器窗口.md` — Unity 编辑器三种窗口类型：ScriptableWizard / EditorWindow / PopupWindowContent

## 要点

1. **ScriptableWizard** — 向导窗口，内置 Create/Apply/Cancel 按钮，`OnWizardUpdate` 验证、`helpString`/`errorString` 提示
2. **EditorWindow** — 完全自定义窗口，支持 IMGUI（`OnGUI`）或 UI Toolkit（`CreateGUI`+UXML/USS），可停靠到编辑器
3. **PopupWindowContent** — 轻量级弹窗，点击外部自动关闭，用于临时 UI
4. **EditorWindow 生命周期** — 10 个消息方法：`OnEnable`/`OnDisable`/`OnGUI`/`Update`（~10fps）/`OnInspectorUpdate`（~50fps）/`OnFocus`/`OnSelectionChange`/`OnProjectChange`/`OnHierarchyChange`
5. **窗口管理** — `GetWindow<T>()` 单例获取、`ShowUtility()` 浮动窗口、`minSize`/`maxSize` 尺寸限制

## 关联 Wiki 页面

- [[concepts/Unity编辑器窗口|编辑器窗口]] — 概念页
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor
