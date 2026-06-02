---
title: "Unity 脚本架构与 Gameplay 框架 — 摘要"
type: source-summary
updated: 2026-05-11
source: "raw/gamedev/gameplay/unity-script-architecture.md"
tags: [unity, architecture, gameplay, unitask, dotween, luban]
---

# Unity 脚本架构与 Gameplay 框架

## 来源

`raw/gamedev/gameplay/unity-script-architecture.md` — Unity 项目脚本架构、架构模式、异步与动画框架、配置表工具与游戏研发流程的系统梳理

## 要点

1. **模块化与分层** — 按 `Core/Characters/Systems/UI/Utilities` 分层组织脚本，每个脚本单一职责，通过 `asmdef` 隔离模块编译
2. **组件化设计** — 利用 Unity Component-based 思想，`GameObject` 组合多个 `MonoBehaviour` 装配功能，避免 God Class
3. **事件系统解耦** — 通过 `EventManager` + `UnityAction` 实现组件间通信，订阅方无需知晓触发方；大型项目可引入 Event Bus 或 ScriptableObject 事件通道
4. **依赖管理** — 优先 `[SerializeField]` 编辑器赋值（零运行时开销），仅在 `Awake` 中 `GetComponent` 获取自身组件，大型项目引入 Zenject/VContainer DI 容器
5. **数据驱动设计** — `ScriptableObject` 作为配置载体：内存高效、版本可控、策划友好；预计算 Hash 存入 ScriptableObject 避免每帧字符串查找
6. **累加式场景加载** — Additive Scene Loading + `PersistentSingleton`：Gameplay 场景与 UI 场景分离，场景切换时管理器实例不丢失
7. **状态模式** — 使用 `IState` 接口 + 状态类管理角色/游戏状态，替代 `if-else` 膨胀
8. **架构模式 MVC/MVP/MVVM** — MVC 适合小型项目 UI 系统；MVP 使 View 轻薄便于测试；MVVM 通过 ViewModel 实现一对多数据绑定，中大型项目推荐
9. **UniTask 异步框架** — 零 GC 分配的值类型 `UniTask<T>`，主线程安全，可 `try-catch` 异常，支持 `WhenAll` 并发、`CancellationToken` 取消、`SwitchToThreadPool` 线程切换
10. **DOTween 补间动画** — 链式 `DOMove/DOFloat/DOColor` 等 API，`SetEase/SetLoops/SetDelay` 链式配置，`Sequence` 编排串并行动画，`SetId` 批量管理
11. **Luban 配置表** — 一次性加载多表，生成强类型 C# 对象，配置与代码分离，策划修改后重新生成即可
12. **对象池与协程** — 对象池减少 `Instantiate/Destroy` 的 GC 开销；协程基于 `IEnumerator` 在主线程运行，`yield return` 挂起到下一帧/指定条件满足时恢复

## 关联 Wiki 页面

- [[concepts/Unity脚本架构|Unity 脚本架构]] — 概念页
- [[concepts/设计模式-创建型|设计模式-创建型]]
- [[concepts/设计模式-结构型|设计模式-结构型]]
- [[concepts/设计模式-行为型|设计模式-行为型]]
- [[concepts/依赖注入|依赖注入]]
