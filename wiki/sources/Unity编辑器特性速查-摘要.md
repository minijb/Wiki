---
title: "Unity 编辑器特性速查 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/editor-extensions/Unity编辑器特性速查.md"
tags: [unity, editor, attribute]
---

# Unity 编辑器特性速查

## 来源

`raw/gamedev/editor-extensions/Unity编辑器特性速查.md` — Unity 编辑器常用 C# Attribute 速查表

## 要点

1. **属性特性（Inspector 显示控制）** — 17 种：`[Range]`/`[Min]`/`[Max]` 数值限制、`[SerializeField]`/`[SerializeReference]` 序列化控制、`[FormerlySerializedAs]` 重命名保留、`[Header]`/`[Space]`/`[Tooltip]` 排版辅助、`[ContextMenu]`/`[ContextMenuItem]` 菜单等
2. **方法特性（执行时机）** — 5 种：`[DrawGizmo]` 绘制、`[MenuItem]` 菜单、`[InitializeOnLoadMethod]` 编辑器启动、`[RuntimeInitializeOnLoadMethod]` 运行时启动（5 种时机）、`[DllImport]` 原生互操作
3. **类特性（行为控制）** — 15 种：`[Serializable]` 序列化、`[RequireComponent]` 自动添加组件、`[ExecuteAlways]`/`[ExecuteInEditMode]` 编辑器执行、`[CustomEditor]`/`[CustomPropertyDrawer]` 自定义编辑器、`[CreateAssetMenu]` ScriptableObject 创建菜单、`[InitializeOnLoad]` 类级自动初始化等
4. **InitializeOnLoad vs InitializeOnLoadMethod** — 类级在静态构造函数中执行，方法级标记静态方法，效果等价但写法不同
5. **`[SerializeReference]`（Unity 2019.3+）** — 支持接口/抽象类字段的多态序列化，区别于 `[SerializeField]` 仅序列化具体值类型

## 关联 Wiki 页面

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 概念页
- [[sources/Unity编辑器全局-摘要|Unity 编辑器全局设置]] — 编辑器文件夹、MenuItem
- [[sources/Unity自定义Inspector-摘要|Unity 自定义 Inspector]] — CustomEditor
- [[sources/Unity自定义PropertyDrawer-摘要|Unity 自定义 PropertyDrawer]] — CustomPropertyDrawer
