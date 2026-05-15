---
title: Unity Gizmos 调试
date: 2026-03-16
tags:
  - unity
  - editor
  - gizmos
  - debug
type: framework
aliases:
  Gizmos
description: Unity Gizmos辅助调试绘制
status: archived
draft: false
archived-to: raw/gamedev/editor-extensions/Unity Gizmos调试.md
---

## 概述

本页介绍 Unity Scene 视图中的调试绘制工具：`OnDrawGizmos` / `[DrawGizmo]`（纯显示）和 `Handles`（可交互）。从快速原型到正式项目的完整对比。

## OnDrawGizmos（MonoBehaviour 内置方法）

```csharp
public class Example : MonoBehaviour
{
    public float radius = 1f;

    // 一直显示（非选中时也绘制）
    private void OnDrawGizmos()
    {
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireSphere(transform.position, radius);
    }

    // 仅选中时显示
    private void OnDrawGizmosSelected()
    {
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, radius + 0.2f);
    }
}
```

| 方法 | 调用时机 |
|------|----------|
| `OnDrawGizmos()` | 每帧，对所有可见且 activeInHierarchy 为 true 的对象 |
| `OnDrawGizmosSelected()` | 每帧，仅选中对象 |

> Hierarchy 中折叠的对象不会触发绘制（因 GameObject 被禁用时不在活动层级中）。Prefab Mode 中同样有效。

**优点**：简单直观，逻辑和数据在同一处
**缺点**：耦合在 MonoBehaviour 中，无法为内置组件绘制；Subscene/Live Link 场景中可能引发不必要的转换

## DrawGizmo（Editor 脚本，推荐）

```csharp
// 放在 Editor 文件夹中
public class ExampleGizmoDrawer
{
    [DrawGizmo(GizmoType.Active | GizmoType.Selected | GizmoType.NonSelected)]
    private static void DrawGizmo(Example target, GizmoType gizmoType)
    {
        Gizmos.color = (gizmoType & GizmoType.Selected) != 0 ? Color.red : Color.yellow;
        Gizmos.DrawWireSphere(target.transform.position, target.radius);
    }
}
```

**优点**：
- Editor 与 Runtime 代码完全分离，build 中零残留
- 可为任意类型（包括 Unity 内置组件）绘制 Gizmos
- 通过 `GizmoType` 位掩码精细控制
- 脚本折叠不影响显示

**对比**：

| 场景 | 推荐方式 |
|------|----------|
| 快速原型/个人项目 | `OnDrawGizmos` |
| 正式项目/团队协作 | `[DrawGizmo]` |
| DOTS / Subscene / Live Link | `[DrawGizmo]`（必须） |
| 为内置组件绘制 | `[DrawGizmo]`（唯一选择） |

## GizmoType 枚举

| 值 | 说明 |
|----|------|
| `Active` | 对象 activeInHierarchy 为 true 时绘制（对普通 GameObject 通常始终为 true） |
| `Selected` | 对象被选中时绘制 |
| `NonSelected` | 对象未被选中时绘制 |
| `InSelectionHierarchy` | 在选中层级的子级中 |
| `Pickable` | Gizmo 可被点击选中 |

## Gizmos 常用方法

> `Gizmos.color` 是全局静态变量，修改后影响后续所有绘制，用完后务必恢复。

```csharp
// 推荐：保存并恢复颜色
var oldColor = Gizmos.color;
Gizmos.color = Color.yellow;
Gizmos.DrawWireSphere(pos, 1f);
Gizmos.color = oldColor;
```

```csharp
// 基础形状
Gizmos.DrawCube(center, size);
Gizmos.DrawWireCube(center, size);
Gizmos.DrawSphere(center, radius);
Gizmos.DrawWireSphere(center, radius);

// 线
Gizmos.DrawLine(from, to);
Gizmos.DrawRay(ray);              // 或 DrawRay(origin, direction)

// 图标（素材需放在 Gizmos 文件夹中）
Gizmos.DrawIcon(position, "iconName.png", allowScaling: true);

// 视锥体（调试摄像机）
Gizmos.DrawFrustum(center, fov, maxRange, minRange, aspect);

// 网格（调试地形/区域）
Gizmos.DrawMesh(mesh, position, rotation, scale);
Gizmos.DrawWireMesh(mesh, position, rotation, scale);

// 纹理
Gizmos.DrawGUITexture(screenRect, texture);
```

> `Gizmos` 在 `UnityEngine` 命名空间中，Runtime 代码也可调用但仅在 Scene 视图可见。

## Handles — 可交互的编辑器绘制

`Handles` 提供比 `Gizmos` 更强大的 3D 绘制能力，**支持交互**（拖拽、旋转等），仅在 Scene 视图可见。`Handles` 在 `UnityEditor` 命名空间中，必须在 Editor 文件夹内使用。

```csharp
// 放在 Editor 文件夹中
[CustomEditor(typeof(Example))]
public class ExampleEditor : Editor
{
    private void OnSceneGUI()
    {
        var t = target as Example;
        var pos = t.transform.position;

        // 绘制（不可交互）
        Handles.color = new Color(1, 0.8f, 0.4f, 1);
        Handles.DrawWireDisc(pos, Vector3.up, 1f);
        Handles.Label(pos + Vector3.up * 2, $"<color=yellow>Value: {t.value:F1}</color>");

        // 交互手柄
        EditorGUI.BeginChangeCheck();
        var newPos = Handles.PositionHandle(pos, Quaternion.identity);
        if (EditorGUI.EndChangeCheck())
        {
            Undo.RecordObject(t, "Move Example");
            t.transform.position = newPos;
        }
    }
}
```

> `Handles.Label` 支持富文本标签（`<color>`、`<b>`、`<i>`、`<size>`）。

| Handles 方法 | 说明 |
|-------------|------|
| `PositionHandle(pos, rot)` | 3D 位置拖拽手柄 |
| `RotationHandle(rot, pos)` | 3D 旋转手柄 |
| `ScaleHandle(scale, pos, rot, size)` | 3D 缩放手柄 |
| `Slider(pos, dir)` | 1D 滑动手柄 |
| `FreeMoveHandle(pos, size, snap, capFunc)` | 自由移动手柄 |
| `Button(pos, rot, size, pickSize, capFunc)` | 3D 可点击按钮 |
| `DrawLine(p1, p2)` | 绘制线段 |
| `DrawAAPolyLine(points)` | 抗锯齿线段（比 `DrawLine` 更清晰） |
| `DrawWireDisc(center, normal, radius)` | 绘制圆环 |
| `DrawSolidDisc(center, normal, radius)` | 绘制实心圆盘 |
| `DrawWireArc(center, normal, from, angle, radius)` | 绘制弧形 |
| `DrawSolidArc(center, normal, from, angle, radius)` | 绘制扇形 |
| `Label(position, text)` | 3D 文字标签（支持富文本） |

### Gizmos vs Handles

| | Gizmos | Handles |
|------|--------|---------|
| **命名空间** | `UnityEngine`（Runtime 可用） | `UnityEditor`（仅 Editor） |
| **交互** | ❌ 纯显示 | ✅ 可拖拽/旋转/点击 |
| **可见范围** | Scene + Game 视图 | 仅 Scene 视图 |
| **脚本位置** | 任意 MonoBehaviour 或 Editor | 必须 Editor 文件夹 |
| **用途** | 调试可视化 | 编辑器工具/自定义手柄 |

> 所有 Gizmos/Handles 绘制都应避免复杂计算 — 每帧调用会直接影响编辑器流畅度。

## 参见

- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹、MenuItem、Selection
- [[concepts/Unity自定义Inspector|自定义 Inspector]] — CustomEditor + OnSceneGUI
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表
- [[concepts/Unity编辑器窗口|编辑器窗口]] — EditorWindow / ScriptableWizard
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer
