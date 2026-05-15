---
title: "Unity 自定义 Inspector — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/editor-extensions/Unity自定义Inspector.md"
tags: [unity, editor, inspector, custom-editor]
---

# Unity 自定义 Inspector

## 来源

`raw/gamedev/editor-extensions/Unity自定义Inspector.md` — 通过 `[CustomEditor]` 自定义 MonoBehaviour 的 Inspector 面板

## 要点

1. **两种实现方式对比** — 直接访问 `target` 简单但不支持自动 Undo；`SerializedProperty` 自动支持 Undo/Prefab 覆盖/多选
2. **SerializedProperty（推荐）** — `OnEnable` 中 `FindProperty`、`OnInspectorGUI` 中 `Update` + `PropertyField` + `ApplyModifiedProperties`
3. **布局控件** — `BeginHorizontal`/`EndHorizontal`、`Foldout`、`HelpBox`、`GUI.enabled`、`EditorGUI.indentLevel`
4. **数组处理** — `PropertyField(prop, true)` 自动绘制可增删数组，手动操作 `InsertArrayElementAtIndex`/`DeleteArrayElementAtIndex`
5. **OnSceneGUI** — CustomEditor 可在 Scene 视图绘制，通过 `Handles.Label` 等显示调试信息

## 关联 Wiki 页面

- [[concepts/Unity自定义Inspector|自定义 Inspector]] — 概念页
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — Handles / OnSceneGUI
