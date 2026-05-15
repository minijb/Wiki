---
title: "Unity 编辑器全局设置 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/editor-extensions/Unity编辑器全局.md"
tags: [unity, editor, menu-item, selection, context-menu]
---

# Unity 编辑器全局设置

## 来源

`raw/gamedev/editor-extensions/Unity编辑器全局.md` — Unity 编辑器扩展基础机制速查

## 要点

1. **Editor 文件夹** — Unity 通过命名约定自动识别名为 `Editor` 的文件夹，其内脚本仅在编辑器编译，构建时排除。可放在任意位置、可存在多个
2. **Editor Default Resources** — 必须放 Assets 根目录下，存编辑器资源，通过 `EditorGUIUtility.Load()` 加载，不打包
3. **Gizmos 文件夹** — 放 Gizmos 图标资源，通过 `Gizmos.DrawIcon()` 使用，不打包
4. **MenuItem** — 添加菜单栏按钮，支持优先级分栏（差值 > 10 出现分隔线）、快捷键（%/#/& 符号）、Validate 验证函数
5. **CONTEXT** — 为组件 Inspector 标题栏右键菜单添加选项，格式 `CONTEXT/组件类型名/菜单名`
6. **MenuCommand** — CONTEXT 菜单方法参数，`cmd.context` 获取当前组件引用。普通 MenuItem 中为 null
7. **ContextMenu** — 为组件齿轮菜单添加选项，方法须 `public`、在同一个类中
8. **ContextMenuItem** — 为字段右键菜单添加选项，第一个参数菜单名、第二个回调方法名
9. **Selection** — 访问编辑器选中对象（`activeGameObject`、`objects`、`assetGUIDs`、`selectionChanged` 事件等），支持 `GetFiltered<T>()` 类型过滤

## 关联 Wiki 页面

- [[concepts/Unity编辑器全局设置|Unity 编辑器全局设置]] — 概念页
- [[sources/Unity编辑器特性速查-摘要|Unity 编辑器特性速查]] — 内置 Attribute 速查
- [[sources/Unity自定义PropertyDrawer-摘要|Unity 自定义 PropertyDrawer]] — 自定义 PropertyAttribute
- [[sources/Unity自定义Inspector-摘要|Unity 自定义 Inspector]] — CustomEditor
- [[sources/Unity编辑器窗口-摘要|Unity 编辑器窗口]] — EditorWindow / ScriptableWizard
- [[sources/Unity Gizmos调试-摘要|Unity Gizmos 调试]] — Gizmos / Handles
