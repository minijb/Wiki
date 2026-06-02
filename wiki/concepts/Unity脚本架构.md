---
title: "Unity 脚本架构"
type: concept
updated: 2026-05-11
tags: [unity, architecture, gameplay, unitask, dotween, luban]
---

# Unity 脚本架构

Unity 游戏开发中，脚本架构决定了项目的可维护性、可扩展性与团队协作效率。核心原则包括高内聚低耦合、开闭原则、避免 God Class。本文覆盖项目架构设计、架构模式（MVC/MVP/MVVM）、异步框架（UniTask）、动画引擎（DOTween）、配置表工具（Luban）、游戏研发流程以及对象池与协程。

## 项目架构设计

清晰的 Unity 项目架构是可持续迭代的基础。

### 模块化与分层

**目录分层**：`Core/`（全局管理器、事件中心）、`Characters/`（角色 Controller/Inventory/Stats）、`Systems/`（存档、对话、任务子系统）、`UI/`（界面管理）、`Utilities/`（扩展方法、单例基类）。

**组件化**：利用 Unity Component-based 思想，每个 `MonoBehaviour` 单一职责（如 `PlayerMovement` 仅处理移动），通过 `GameObject` 组合装配功能。

**工具集成**：使用 `.asmdef` 隔离模块编译，搭配 Roslynator/Rider 进行静态代码分析。

### 事件系统与通信

组件间通过 `EventManager` + `UnityAction` 解耦，而非直接引用。订阅方在 `OnEnable` 订阅事件，在 `OnDisable` 取消订阅。大型项目可引入 Event Bus 或 ScriptableObject 事件通道消除静态依赖。

### 依赖管理

- **`[SerializeField]` 编辑器拖拽**：编译时确定引用，零运行时开销，首选方案
- **`GetComponent` 仅在 `Awake` 获取自身组件**：如 `GetComponent<Rigidbody>()`
- **DI 容器**：大型项目引入 Zenject/Extenject 或 VContainer 管理对象生命周期

### 数据驱动设计

`ScriptableObject` 作为配置载体：内存高效（数据共享同一实例）、版本可控（`.asset` 文件可纳入 VCS）、策划友好（Inspector 直接编辑）。数据实体化策略：将动画名称、音效 ID 等预先计算 Hash 存入 ScriptableObject，避免运行时字符串查找。

### 场景管理

累加式场景加载 + `PersistentSingleton`：核心 Gameplay 场景与 UI 场景分离，UI 可在不卸载游戏场景的情况下独立重载。

## 架构模式

### MVC（Model-View-Controller）

- **Model**：纯数据（`ScriptableObject` 或 C# 类），负责数据存取与验证
- **View**：MonoBehaviour 展示层，渲染数据到屏幕
- **Controller**：协调 Model 与 View，处理业务逻辑

数据流：`View 触发事件 → Controller 处理 → 更新 Model → View 刷新`

### MVP（Model-View-Presenter）

View 与 Model 完全隔离。Presenter 持有 View 引用并直接调用其更新方法。View 极端轻薄，便于单元测试。

### MVVM（Model-View-ViewModel）

ViewModel 替代 Presenter，与 View 为一对多关系。通过数据绑定驱动 View 更新，可借助 UniRx 或 `INotifyPropertyChanged` 实现响应式。

> 选择建议：小型项目用 MVC，中大型项目推荐 MVVM 或 ECS。

## UniTask 异步

[UniTask](https://github.com/Cysharp/UniTask) 是 Unity 生态最广泛使用的异步编程库。

### 核心优势

| | 协程 | async Task | UniTask |
|---|---|---|---|
| try-catch | 不支持 | 支持 | 支持 |
| 返回值 | 不支持 | 支持 | 支持 |
| GC 分配 | 有 | 有（引用类型） | 零（值类型） |
| 主线程安全 | 是 | 否（默认线程池） | 是 |

### 常用 API

- **时间控制**：`UniTask.Yield()`、`UniTask.Delay()`、`UniTask.DelayFrame()`、`UniTask.WaitForEndOfFrame()`
- **条件等待**：`UniTask.WaitUntil()`、`UniTask.WaitUntilValueChanged()`
- **并发**：`UniTask.WhenAll(t1, t2, t3)`，支持元组解构
- **取消**：`CancellationToken` + `this.GetCancellationTokenOnDestroy()`
- **线程切换**：`UniTask.SwitchToThreadPool()` / `SwitchToMainThread()`

> 严禁同一 `UniTask` 实例多次 `await`，操作重复消费将抛出异常。

## DOTween 动画

[DOTween](http://dotween.demigiant.com/) 是 Unity 最流行的补间动画库，通过链式调用实现流畅的代码动画。

### 基础用法

```csharp
transform.DOMoveX(5f, 1f).SetEase(Ease.OutBounce);
DOTween.To(() => currentValue, x => currentValue = x, targetValue, 2f);
transform.DOMove(target, 1f).From(true); // 相对 From
```

### 链式配置

`SetDelay`、`SetEase`（30+ 曲线）、`SetLoops`（Restart/Yoyo/Incremental）、`SetAutoKill`、`SetUpdate`（忽略 timeScale）、`SetId`（批量管理）

### Sequence 序列

`Append`（顺序添加）、`Join`（并行动画）、`Insert`（指定时间点插入）、`Prepend`（插入开头）。生命周期回调：`OnStart/OnComplete/OnKill/OnPause`。

## Luban 配置表

[Luban](https://github.com/focus-creative-games/luban) 是 focal-creative-games 开源的游戏配置表方案。一次性加载多表，生成强类型 C# 对象，配置与代码完全分离。策划修改配置表后重新运行 Luban 生成工具即可更新类型定义。

```csharp
Tables tables = new Tables(LoadTable);
// LoadTable 回调中从 Resources 读取 JSON
```

## 游戏研发流程

| 阶段 | 目标 | 关键活动 |
|---|---|---|
| 调查立项 | 验证市场可行性 | 竞品分析、题材确立、成本预估 |
| 原型阶段 | 验证玩法可玩性 | 基础框架与核心玩法实现 |
| Alpha 阶段 | 丰富血肉 | 宏观设计、3D 建模、动画、音效 |
| Beta 阶段 | 测试稳定 | 功能测试与迭代修复 |
| 发行阶段 | Demo 对接渠道 | 发行运营介入 |
| 上市/运营 | 维护迭代 | 用户反馈驱动更新、多版本移植 |

团队六大职能：策划（文案/数值/系统/战斗）、美术（2D/3D/动画/特效/TA）、技术（客户端/服务器/QA/PM）、发行运营。

## 对象池与协程

- **对象池**：预先实例化对象入池，需要时取出激活，用完后重置放回，避免频繁 `Instantiate/Destroy` 的 GC 开销。典型场景：子弹、粒子特效、UI 列表项。
- **协程**：基于 `IEnumerator` 的迭代器模式，`yield return` 挂起到下一帧/指定条件满足，运行在主线程。`SetActive(false)` 会停止协程，`enabled = false` 不会。
- **生命周期**：`Awake → OnEnable → Start → Update → FixedUpdate → LateUpdate → OnGUI → OnDisable → OnDestroy`

## 参见

- [[sources/Unity脚本架构-摘要|来源摘要]]
- [[concepts/设计模式-创建型|设计模式-创建型]]
- [[concepts/设计模式-结构型|设计模式-结构型]]
- [[concepts/设计模式-行为型|设计模式-行为型]]
- [[concepts/依赖注入|依赖注入]]
- [[concepts/CSharp异步模型|CSharp 异步模型]] — async/await 状态机与 Unity 主线程调度
