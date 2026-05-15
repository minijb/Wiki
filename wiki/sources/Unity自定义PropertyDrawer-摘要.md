---
title: "Unity 自定义 PropertyDrawer — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/editor-extensions/Unity自定义PropertyDrawer.md"
tags: [unity, editor, property-drawer, attribute]
---

# Unity 自定义 PropertyDrawer

## 来源

`raw/gamedev/editor-extensions/Unity自定义PropertyDrawer.md` — 通过自定义 PropertyAttribute + PropertyDrawer 扩展 Inspector 字段显示

## 要点

1. **C# 自定义 Attribute 是基础** — 定义 Attribute 类 + 反射读取，Unity 的 `[CustomEditor]`、`[CustomPropertyDrawer]` 等均基于此机制
2. **`[AttributeUsage]`** 限制作用目标（类、字段、方法），实战建议始终声明
3. **定义 PropertyAttribute** — 继承 `PropertyAttribute`，可添加自定义参数。`order` 属性控制多个 Drawer 的执行顺序
4. **定义 PropertyDrawer** — 继承 `PropertyDrawer`，用 `[CustomPropertyDrawer(typeof(Attribute))]` 标记，覆写 `OnGUI` 和 `GetPropertyHeight`
5. **`attribute` 属性** — 在 `OnGUI` 中通过 `attribute as XXXAttribute` 获取自定义参数
6. **完整示例**：`ShowTimeAttribute` 将秒数显示为 `HH:MM:SS` 格式，上半行整数输入 + 下半行时间预览

## 关联 Wiki 页面

- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — 概念页
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor
- [[sources/Unity编辑器特性速查-摘要|Unity 编辑器特性速查]] — 来源摘要
