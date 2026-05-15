---
title: "Unity Gizmos 调试 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/editor-extensions/Unity Gizmos调试.md"
tags: [unity, editor, gizmos, handles, debug]
---

# Unity Gizmos 调试

## 来源

`raw/gamedev/editor-extensions/Unity Gizmos调试.md` — Unity Scene 视图调试绘制：Gizmos（纯显示）与 Handles（可交互）

## 要点

1. **OnDrawGizmos** — MonoBehaviour 内置方法，简单直观但耦合在组件中。`OnDrawGizmos()` 始终绘制、`OnDrawGizmosSelected()` 仅选中时绘制
2. **DrawGizmo（推荐）** — Editor 脚本中通过 `[DrawGizmo(GizmoType)]` 声明，Editor/Runtime 分离，可为内置组件绘制，DOTS/Subscene 必须使用
3. **GizmoType** — 位掩码精细控制：`Active`、`Selected`、`NonSelected`、`InSelectionHierarchy`、`Pickable`
4. **Gizmos 常用绘制** — 立方体/球/线/射线/图标/视锥/网格/纹理，所有绘制应避免复杂计算影响帧率
5. **颜色管理** — `Gizmos.color` 是全局静态变量，使用前后须保存恢复：`var old = Gizmos.color; ... Gizmos.color = old;`
6. **Handles** — 可交互的编辑器绘制，支持拖拽/旋转（`PositionHandle`/`RotationHandle`），仅在 Scene 视图可见，必须在 Editor 文件夹中使用
7. **命名空间差异** — `Gizmos` 在 `UnityEngine`（Runtime 可用），`Handles` 在 `UnityEditor`（仅 Editor）
8. **Handle 常用 API** — 交互手柄（Position/Rotation/Scale/Slider/Button）、绘制（Line/AAPolyLine/WireDisc/WireArc/Label 支持富文本）

## 关联 Wiki 页面

- [[concepts/Unity Gizmos 调试|Gizmos 调试]] — 概念页
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — OnSceneGUI 配合 Handles
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹
