---
title: "Unity Gizmos 调试"
type: concept
updated: 2026-05-11
tags: [unity, editor, gizmos, handles, debug]
---

# Unity Gizmos 调试

Unity Scene 视图中的调试绘制工具：Gizmos（纯可视化）与 Handles（可交互）。

## Gizmos — 纯显示

### OnDrawGizmos（内置于组件）

```csharp
private void OnDrawGizmos()          // 始终绘制
private void OnDrawGizmosSelected()  // 仅选中时
```

简单直观但耦合在 MonoBehaviour 中，无法为内置组件绘制。

### DrawGizmo（Editor 脚本，推荐）

```csharp
[DrawGizmo(GizmoType.Active | GizmoType.Selected | GizmoType.NonSelected)]
private static void DrawGizmo(MyComponent target, GizmoType gizmoType) { }
```

- Editor/Runtime 分离，build 零残留
- GizmoType 位掩码精细控制
- DOTS/Subscene/Live Link 必须使用

**GizmoType 枚举**：`Active`（对象激活时）、`Selected`、`NonSelected`、`InSelectionHierarchy`、`Pickable`

### Gizmos vs DrawGizmo 选择

| 场景 | 推荐 |
|------|------|
| 快速原型 | `OnDrawGizmos` |
| 正式项目/团队 | `[DrawGizmo]` |
| 为内置组件绘制 | `[DrawGizmo]`（唯一） |
| DOTS/Subscene | `[DrawGizmo]`（必须） |

### 颜色管理

```csharp
var oldColor = Gizmos.color;
Gizmos.color = Color.yellow;
// ... 绘制 ...
Gizmos.color = oldColor; // 必须恢复
```

### 常用绘制

`DrawCube` / `DrawWireCube` / `DrawSphere` / `DrawWireSphere` / `DrawLine` / `DrawRay` / `DrawIcon` / `DrawFrustum` / `DrawMesh` / `DrawWireMesh`

## Handles — 可交互

`Handles` 在 `UnityEditor` 命名空间，必须放 Editor 文件夹，仅 Scene 视图可见，支持拖拽/旋转/点击。

### 交互手柄

| API | 说明 |
|-----|------|
| `PositionHandle(pos, rot)` | 3D 位置拖拽 |
| `RotationHandle(rot, pos)` | 3D 旋转 |
| `ScaleHandle(scale, pos, rot, size)` | 3D 缩放 |
| `Slider(pos, dir)` | 1D 滑动 |
| `Button(pos, rot, size, pickSize, capFunc)` | 3D 可点击按钮 |

### 配合 Undo

```csharp
EditorGUI.BeginChangeCheck();
var newPos = Handles.PositionHandle(pos, Quaternion.identity);
if (EditorGUI.EndChangeCheck())
{
    Undo.RecordObject(target, "Move");
    ((MyComponent)target).transform.position = newPos;
}
```

### 绘制 API

`DrawLine` / `DrawAAPolyLine`（抗锯齿）/ `DrawWireDisc` / `DrawSolidDisc` / `DrawWireArc` / `DrawSolidArc` / `Label`（支持 `<color>` 等富文本）

## Gizmos vs Handles

| | Gizmos | Handles |
|---|---|---|
| **命名空间** | `UnityEngine` | `UnityEditor` |
| **交互** | ❌ 纯显示 | ✅ 可拖拽/旋转 |
| **可见范围** | Scene + Game | 仅 Scene |
| **脚本位置** | 任意或 Editor | 必须 Editor 文件夹 |

## 参见

- [[concepts/Unity自定义Inspector|自定义 Inspector]] — OnSceneGUI 配合 Handles
- [[concepts/Unity编辑器全局设置|编辑器全局设置]] — Editor 文件夹
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — `[DrawGizmo]` 特性
- [[sources/Unity Gizmos调试-摘要|来源摘要]]
