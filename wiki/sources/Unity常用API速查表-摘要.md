---
title: "Unity 常用 API 速查表 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/gamedev/gameplay/unity-api-cheatsheet.md"
tags: [unity, api, cheatsheet, editor, ui, transform, input, serialization]
---

# Unity 常用 API 速查表

## 来源

`raw/gamedev/gameplay/unity-api-cheatsheet.md` — 合并自 40+ 条 Unity 杂项笔记的速查表

## 要点

1. **Transform/GameObject** — 基本关系（不能互转但互取）、位移（Translate/手动/forward 方向）、父子关系（SetParent/Find/GetChild/兄弟操作）
2. **Time 与帧率** — timeScale/deltaTime/unscaledDeltaTime/fixedDeltaTime、timeScale 对 FixedUpdate/Coroutine 的影响、targetFrameRate
3. **Input System** — 旧版 GetKey/GetKeyDown/GetKeyUp；新版 InputSystem（Action Type: Value/Button/PassThrough、事件订阅）
4. **UI 系统** — Canvas Scaler（Scale With Screen Size/Expand/Shrink）、Content Size Fitter、锚点/Pivot、判断点是否在 UI 内（Camera + Overlay 两种方案）、ScreenPointToLocalPointInRectangle
5. **UGUI 底层** — OnPopulateMesh 顶点生成、CanvasUpdateRegistry Rebuild 流程、OverDraw 优化（Cull Transparent Mesh / RectMask2D）
6. **序列化** — SerializeField/Serializable、ScriptableObject（CreateAssetMenu/用途/Build 后注意）、SerializedProperty、PropertyAttribute/PropertyDrawer、AutoHook 自动绑定
7. **常用模式** — 单例（饿汉/懒汉）、闭包GC问题（原因与优化）、struct 栈分配实验、UniTask 线程技巧
8. **资源与内存** — material/sharedMaterial 区别、MaterialPropertyBlock、AssetBundle Unload(true/false)、LZ4 vs LZMA
9. **序列化格式** — JsonUtility / System.Text.Json、Protobuf（序列化/反序列化/反射通用解码）、byte[] 拼接工具
10. **编辑器扩展** — Editor文件夹/Gizmos文件夹、MenuItem（快捷键/验证/优先级/右键菜单）、ContextMenu/ContextMenuItem、Selection 类、AssetDatabase、Mono.Cecil、Editor 按钮变色、脚本宏管理、ParrelSync
11. **面试精华** — FSM 实现（状态/转换/FSMSystem）、行为树（Sequencer/Selector）、三消算法思路、柏林噪声原理、子弹穿透检测、多相机叠加（depth/Depth Only）、事件中心、List 内部实现

## 关联 Wiki 页面

- [[concepts/Unity编辑器特性速查|Unity 编辑器特性速查]]
- [[concepts/Unity自定义Inspector|自定义 Inspector]]
- [[concepts/Unity自定义PropertyDrawer|自定义 PropertyDrawer]]
- [[concepts/Unity编辑器窗口|编辑器窗口]]
- [[concepts/Unity编辑器全局设置|编辑器全局设置]]
- [[concepts/CSharp值类型性能|C# 值类型性能]]
- [[concepts/CSharp集合框架|C# 集合框架]]
