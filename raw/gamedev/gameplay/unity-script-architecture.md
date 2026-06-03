---
title: "Unity 脚本架构与 Gameplay 框架"
type: source
updated: 2026-06-02
tags:
  - unity
  - architecture
  - gameplay
  - unitask
  - dotween
  - luban
---

# Unity 脚本架构与 Gameplay 框架

Unity 游戏开发中，脚本架构（Script Architecture）决定了项目的可维护性、可扩展性与团队协作效率。本文从项目架构设计、架构模式（MVC/MVP/MVVM）、异步框架（UniTask）、动画引擎（DOTween）、配置表工具（Luban）、游戏研发流程以及对象池与协程等核心主题出发，系统梳理 Unity Gameplay 开发中的关键设计决策与技术选型。

## 项目架构设计

一个清晰的 Unity 项目架构是可持续迭代的基础。核心原则包括：**高内聚低耦合**（模块内部紧密关联，模块间依赖最小化）、**开闭原则**（对扩展开放，对修改关闭）、**避免 God Class**（拒绝万能管理器，超过 500 行的类强制拆分）。

### 模块化与分层

**模块化设计**要求每个脚本只负责单一功能。例如 `PlayerMovement` 仅处理移动、`EnemyAI` 仅处理 AI 逻辑。利用 Unity 组件化思想（Component-based），通过 `GameObject` 组合多个 `MonoBehaviour` 来装配功能，而非在一个类中堆砌所有逻辑。

```csharp
// 独立移动组件 — 单一职责
public class CharacterMovement : MonoBehaviour
{
    [SerializeField] private float moveSpeed = 5f;

    public void Move(Vector2 direction)
    {
        transform.Translate(direction * moveSpeed * Time.deltaTime);
    }
}
```

**分层架构**将脚本按职责归类到不同命名空间与目录：

```
Assets/
├─ Scripts/
│  ├─ Core/           // 核心系统（GameManager, EventSystem）
│  ├─ Characters/     // 角色相关（Player/, NPC/）
│  ├─ Systems/        // 子系统（SaveSystem, DialogueSystem）
│  ├─ UI/             // 界面系统（UIManager, HUD/）
│  ├─ Utilities/      // 工具类（Extensions, Singleton）
```

- **Core/**：存放全局管理器、事件中心、场景加载器等基础设施。这些类通常以单例（Singleton）或持久单例（PersistentSingleton）形式存在。
- **Characters/**：按角色类型（Player、NPC、Enemy）细分，每个角色拥有各自的 Controller、Inventory、Stats 等组件。
- **Systems/**：独立子系统（如存档、对话、任务），与角色和 UI 解耦。
- **UI/**：界面管理与 HUD 组件，通过事件系统与数据层交互。
- **Utilities/**：扩展方法、泛型单例基类、数学工具等零依赖工具。

**设计模式应用**：状态模式（State Pattern）常用于管理角色状态，避免 `if-else` 膨胀：

```csharp
public interface IPlayerState
{
    void EnterState(PlayerController player);
    void Update();
    void ExitState();
}

public class RunningState : IPlayerState
{
    public void EnterState(PlayerController player)
    {
        player.Animator.Play("Run");
    }

    public void Update()
    {
        // 检测输入切换状态
    }

    public void ExitState() { }
}
```

**工具集成**：使用 Assembly Definition（`.asmdef`）文件隔离模块编译，减少增量编译时间。搭配 Roslynator 或 JetBrains Rider 进行静态代码分析，强制执行命名规范与代码质量门禁。

### 事件系统与通信

组件间通信应通过**事件系统**解耦，而非直接引用。典型实现是以 `UnityAction`（或 `UnityEvent`）为基础的 `EventManager`：

```csharp
public static class EventManager
{
    public static UnityAction OnPlayerDeath;
    public static UnityAction<int> OnScoreChanged;
    public static UnityAction<ItemData> OnItemCollected;

    public static void TriggerPlayerDeath() => OnPlayerDeath?.Invoke();
    public static void TriggerScoreChanged(int newScore) => OnScoreChanged?.Invoke(newScore);
    public static void TriggerItemCollected(ItemData item) => OnItemCollected?.Invoke(item);
}

// 使用端 — 订阅方无需知晓触发方
public class UIManager : MonoBehaviour
{
    private void OnEnable()
    {
        EventManager.OnScoreChanged += UpdateScoreDisplay;
    }

    private void OnDisable()
    {
        EventManager.OnScoreChanged -= UpdateScoreDisplay;
    }

    private void UpdateScoreDisplay(int score) { /* 更新 UI */ }
}
```

事件系统带来的收益：新功能模块只需订阅已有事件即可接入，无需修改现有代码；单元测试可通过直接触发事件来模拟游戏状态变化。

对于更复杂的场景，可引入 **Event Bus** 或基于 `ScriptableObject` 的事件通道（Event Channel），进一步消除静态依赖。

### 依赖管理

**避免 `GetComponent()` 滥用**是 Unity 性能优化的常备条目。优先通过以下方式获取引用：

1. **`[SerializeField]` 编辑器拖拽赋值**：编译时确定引用，零运行时开销。
2. **`GetComponent` 仅在 `Awake` 中获取自身组件**：如 `GetComponent<Rigidbody>()`，避免跨对象查找。
3. **依赖注入（DI）模式**：通过构造函数或属性注入依赖，便于测试与替换实现。

```csharp
public class PlayerCombat : MonoBehaviour
{
    // 编辑器拖拽 — 最优
    [SerializeField] private WeaponSystem weapon;

    // Awake 中获取自身组件
    private PlayerStats stats;
    private Rigidbody rb;

    private void Awake()
    {
        stats = GetComponent<PlayerStats>();
        rb = GetComponent<Rigidbody>();
    }
}
```

对于大型项目，可引入 DI 容器（如 Zenject/Extenject 或 VContainer）管理对象生命周期与依赖图。

### 数据驱动设计

使用 `ScriptableObject` 作为游戏配置与数据的载体，具有以下优势：

- **内存高效**：数据在编辑器和运行时共享同一实例，避免每实例拷贝。
- **版本可控**：`.asset` 文件可纳入版本控制，支持多人协作。
- **策划友好**：策划可在 Inspector 中直接编辑数据，无需接触代码。

```csharp
[CreateAssetMenu(fileName = "NewItem", menuName = "Game/ItemData")]
public class ItemData : ScriptableObject
{
    public string itemName;
    public Sprite icon;
    public int maxStack = 1;
    public string description;
    public ItemRarity rarity;
}
```

数据实体化（Data Entity）策略：将动画名称、音效 ID、属性曲线等运行时高频访问的数据预先计算 Hash 并存入 `ScriptableObject`，避免每帧字符串查找。

### 场景管理

采用**累加式场景加载**（Additive Scene Loading）与**持久单例模式**（PersistentSingleton）：

```csharp
public class SceneLoader : PersistentSingleton<SceneLoader>
{
    public void LoadGameScene()
    {
        SceneManager.LoadSceneAsync("Gameplay");
        SceneManager.LoadSceneAsync("UI", LoadSceneMode.Additive);
    }
}
```

核心场景（Gameplay）与 UI 场景分离，允许 UI 场景在不卸载游戏场景的情况下独立重载。`PersistentSingleton` 确保场景切换时管理器实例不丢失。

**代码规范补充**：接口以 `I` 前缀（如 `IInteractable`），私有字段以 `_` 前缀（如 `_currentHealth`），公共方法必须附带 `/// <summary>` XML 文档注释，标明功能与依赖。

## 架构模式

游戏 UI 系统与业务逻辑的组织方式直接影响代码的可测试性与维护成本。三种主流的 UI 架构模式在游戏开发中各有适用场景。

### MVC（Model-View-Controller）

MVC 将应用分为三层：

- **Model（模型）**：数据层 — 负责数据的存取、验证与持久化。仅关注数据准确性，不涉及展示逻辑。
- **View（视图）**：展示层 — 负责将数据渲染到屏幕，监听用户输入并转发给 Controller。
- **Controller（控制器）**：逻辑层 — 接收 View 的事件，执行业务逻辑，更新 Model 数据。

**数据流**：`View 触发事件 → Controller 处理业务逻辑 → 更新 Model 数据 → Model 写回 View → View 刷新显示`

在 Unity 中，MVC 常用于 UI 系统。Model 可以是 `ScriptableObject` 或纯 C# 数据类，View 是 `UIManager`/`HUD` 等 MonoBehaviour，Controller 负责协调两者。

```csharp
// Model — 纯数据
public class PlayerModel
{
    public int Health { get; set; }
    public int Score { get; set; }
}

// View — 展示
public class PlayerHUDView : MonoBehaviour
{
    [SerializeField] private Text healthText;

    public void UpdateHealth(int health)
    {
        healthText.text = $"HP: {health}";
    }
}

// Controller — 协调
public class PlayerController : MonoBehaviour
{
    private PlayerModel model;
    private PlayerHUDView view;

    public void TakeDamage(int damage)
    {
        model.Health -= damage;
        view.UpdateHealth(model.Health);
    }
}
```

### MVP（Model-View-Presenter）

MVP 是 MVC 的演进，核心差异是 **Presenter 替代 Controller**，且 View 与 Model 完全隔离：

- **View**：仅处理视图渲染，不做任何逻辑处理。将用户交互委托给 Presenter。
- **Model**：完成对数据的操纵 — 网络请求、持久化数据增删改查等均属于 Model 层。
- **Presenter**：作为 View 与 Model 之间的桥梁。持有 View 引用并直接调用其更新方法。

MVP 的优势在于 View 变得极为轻薄，便于 UI 测试（可 Mock View 接口直接测试 Presenter 逻辑）。

### MVVM（Model-View-ViewModel）

MVVM 使用 **ViewModel** 替代 Presenter。关键变化：原本 Presenter 与 View 一对一的关系变为 **ViewModel 与 View 一对多** — 多个 View 可绑定同一个 ViewModel。

- **ViewModel**：封装展示逻辑与状态，通过数据绑定（Data Binding）驱动 View 更新。
- **复用性提升**：ViewModel 被多个 View 共享时，其背后的 Model 也自然被复用。

在 Unity 中，MVVM 可借助 UniRx 或手动实现 `INotifyPropertyChanged` 达到响应式更新。

> 选择建议：小型项目可用基础分层或 MVC；中大型项目推荐 MVVM 或 ECS 架构。

## UniTask 异步框架

[UniTask](https://github.com/Cysharp/UniTask) 是 Unity 生态中最广泛使用的异步编程库，为 C# `async/await` 在 Unity 中的高效运行提供了零 GC 的实现方案。

### 设计动机

Unity 原生的异步方案各有痛点：

| 方案 | 优势 | 痛点 |
|------|------|------|
| `Update()` 轮询 | 简单直接 | 依赖 `MonoBehaviour`，引入大量成员变量与状态标志 |
| 协程（Coroutine） | 语法清晰 | 无法 `try-catch` 异常，依赖 `MonoBehaviour`，无返回值 |
| `async Task` | 标准 C# | 默认使用线程池，跨线程访问 Unity API 导致崩溃，`Task` 是引用类型产生 GC |

UniTask 的解决方案：
- **继承 `async/await` 优势**：可 `try-catch`，可返回值，语法优雅。
- **默认主线程执行**：与 Unity 协同，无跨线程问题。
- **值类型 `UniTask<T>`**：基于 `AsyncMethodBuilder` 实现，零堆分配，无 GC 压力。

### 基础用法

安装方式（Unity Package Manager git URL）：

```
https://github.com/Cysharp/UniTask.git?path=src/UniTask/Assets/Plugins/UniTask
```

**异步资源加载**：直接 `await` Unity 的异步操作即可：

```csharp
// 基础异步加载
var asset = await Resources.LoadAsync<TextAsset>("foo");
var txt = (await UnityWebRequest.Get("https://...").SendWebRequest()).downloadHandler.text;
await SceneManager.LoadSceneAsync("scene2");
```

**时间控制 API**：

```csharp
// 按帧等待 — 替代 yield return null
await UniTask.Yield();
await UniTask.NextFrame();

// 延迟指定帧数
await UniTask.DelayFrame(100);

// 按时间延迟 — 替代 WaitForSeconds
await UniTask.Delay(TimeSpan.FromSeconds(10), ignoreTimeScale: false);

// 在特定 PlayerLoop 阶段执行
await UniTask.Yield(PlayerLoopTiming.PreLateUpdate);

// 替代 WaitForEndOfFrame（Unity 2023.1+ 无需传参）
await UniTask.WaitForEndOfFrame();

// 替代 WaitForFixedUpdate
await UniTask.WaitForFixedUpdate();
```

**条件等待**：

```csharp
// 替代 yield return new WaitUntil(...)
await UniTask.WaitUntil(() => isActive == false);

// 监听字段值变化 — 自动追踪，值改变时恢复
await UniTask.WaitUntilValueChanged(this, x => x.isActive);
```

**扩展方法**：UniTask 为 Unity 的 `AsyncOperation`、`ResourceRequest`、`AssetBundleRequest` 等原生异步类型提供了扩展：

```csharp
// 基础等待
await asyncOperation;

// 绑定取消令牌
await Resources.LoadAsync<TextAsset>("bar")
    .WithCancellation(this.GetCancellationTokenOnDestroy());

// 带进度回调
await Resources.LoadAsync<TextAsset>("baz")
    .ToUniTask(Progress.Create<float>(x => Debug.Log(x)));
```

**UniTaskCompletionSource**：轻量级 `TaskCompletionSource`，用于将回调式 API 包装为 UniTask：

```csharp
public UniTask<int> WrapCallbackAsync()
{
    var utcs = new UniTaskCompletionSource<int>();
    // 完成时调用 utcs.TrySetResult(value);
    // 失败时调用 utcs.TrySetException(ex);
    // 取消时调用 utcs.TrySetCanceled();
    return utcs.Task; // 返回 UniTask<int>
}
```

**类型转换**：`Task` → `UniTask` 用 `.AsUniTask()`，`UniTask<T>` → `UniTask` 用 `.AsUniTask()`（转换成本为零）。支持在 UniTask 中直接 `await` 标准 `Task` 或 `IEnumerator` 协程。

> **严格禁止**：同一 `UniTask` 实例不可 `await` 两次，不可多次调用 `.AsTask()`，不可在操作未完成时使用 `.Result` 或 `.GetAwaiter().GetResult()`。多次消费同一实例将抛出异常。

### 取消与异常

UniTask 使用 .NET 标准的 `CancellationToken` 机制管理异步操作生命周期。

**两种创建 CancellationToken 的方式**：

```csharp
// 方式一：手动创建 CancellationTokenSource
var cts = new CancellationTokenSource();
cancelButton.onClick.AddListener(() => cts.Cancel());
await UnityWebRequest.Get("http://google.co.jp")
    .SendWebRequest()
    .WithCancellation(cts.Token);

// 方式二：绑定 GameObject 生命周期
// CancellationToken 随 GameObject 销毁自动取消
await UniTask.DelayFrame(1000, cancellationToken: this.GetCancellationTokenOnDestroy());
```

**自定义生命周期**：当需要更精细的控制（如 OnEnable 恢复、OnDisable 取消），可自行管理 CancellationToken：

```csharp
public class MyBehaviour : MonoBehaviour
{
    private CancellationTokenSource disableCts;
    private CancellationTokenSource destroyCts = new();

    private void OnEnable()
    {
        disableCts?.Dispose();
        disableCts = new CancellationTokenSource();
    }

    private void OnDisable()
    {
        disableCts?.Cancel();
    }

    private void OnDestroy()
    {
        destroyCts.Cancel();
        destroyCts.Dispose();
    }
}
```

**异常处理**：取消时会抛出 `OperationCanceledException`，未捕获的异常最终传播到 `UniTaskScheduler.UnobservedTaskException`。

```csharp
// 过滤取消异常
public async UniTask<int> BarAsync()
{
    try
    {
        var x = await FooAsync();
        return x * 2;
    }
    catch (Exception ex) when (ex is not OperationCanceledException)
    {
        return -1;
    }
}

// 轻量级忽略取消异常
var (isCanceled, _) = await UniTask.DelayFrame(10, cancellationToken: cts.Token)
    .SuppressCancellationThrow();
if (isCanceled) { /* 处理取消逻辑 */ }
```

**立即取消**：默认情况下 `UniTask.Yield` 与 `UniTask.Delay` 在 PlayerLoop 中检测取消状态，并非立即响应。如需立即取消，传入 `cancelImmediately: true`（代价更高，因为使用了 `CancellationToken.Register`）。

### 线程切换

UniTask 提供的线程切换是其在性能敏感场景下的关键优势：

```csharp
// 切换到线程池 — 执行 CPU 密集型计算
await UniTask.SwitchToThreadPool();
/* 在线程池中执行的代码（不可访问 Unity API）*/

// 切回主线程 — 继续操作 Unity 对象
await UniTask.SwitchToMainThread();
/* 可安全访问 Transform、GameObject 等 Unity API */
```

此模式允许将复杂计算（如寻路、序列化、加密）卸载到后台线程，完成后无缝回到主线程更新 UI 或场景。

### WhenAll 并发

UniTask 支持同时发起多个异步操作，等待全部完成后一次性获取结果：

```csharp
// 并发发起三个网络请求
var task1 = GetTextAsync(UnityWebRequest.Get("http://google.com"));
var task2 = GetTextAsync(UnityWebRequest.Get("http://bing.com"));
var task3 = GetTextAsync(UnityWebRequest.Get("http://yahoo.com"));

// WhenAll 等待所有任务完成，元组解构获取结果
var (google, bing, yahoo) = await UniTask.WhenAll(task1, task2, task3);

// 简写 — 直接 await 元组
var (google2, bing2, yahoo2) = await (task1, task2, task3);
```

`UniTask.WhenAll` 支持最多 15 个泛型参数，`UniTask.WhenAny` 在任一任务完成时返回。返回值可直接使用 C# 元组解构（Tuple Deconstruction），代码简洁且类型安全。

## DOTween 动画引擎

[DOTween](http://dotween.demigiant.com/) 是 Unity 最流行的补间动画（Tweening）库，通过链式调用实现流畅的代码动画，避免手动编写逐帧插值逻辑。

### 基础动画

**最简单的位移动画**：

```csharp
// 1 秒内移动到世界坐标 (5, 0, 0)
transform.DOMoveX(5f, 1f);

// 不受 Time.timeScale 影响（UI 动画刚需）
transform.DOMoveX(5f, 1f).SetUpdate(true);

// 返回 Tween 类型便于后续操作
Tween tween = transform.DOMoveX(5f, 1f);
```

**自定义插值动画** — `DOTween.To()` 可对任意数值类型进行动画：

```csharp
// 参数：(getter, setter, endValue, duration)
DOTween.To(
    () => currentValue,           // 获取当前值
    x => currentValue = x,        // 设置新值
    targetValue,                  // 目标值
    2f                            // 持续时间
);
```

`DOTween.To()` 支持 `float`、`Vector3`、`Color`、`Quaternion` 等常见类型，适用于自定义 Slider 值渐变、材质颜色过渡等场景。

**From 动画**：

```csharp
// 绝对 From：从世界坐标 (2, 2, 0) 回到当前位置
transform.DOMove(new Vector3(2, 2, 0), 1f).From();

// 相对 From（From(true)）：从当前位置 + 偏移 (2, 2, 0) 回到当前位置
transform.DOMove(new Vector3(2, 2, 0), 2f).From(true);
```

**清理残留**：在对象销毁或动画中断时，调用 `.DoKill()` 清理 Tween：

```csharp
private void OnDestroy()
{
    transform.DOKill(); // 杀死该 Transform 上所有动画
}
```

### Set 方法链

DOTween 提供丰富的链式 `Set` 方法，所有方法均返回 `Tween` 类型，支持无限链式组合：

| 方法 | 说明 |
|------|------|
| `SetDelay(float)` | 设置动画延迟（秒） |
| `SetEase(Ease)` | 设置缓动曲线（Linear、InOutQuad、OutBounce、OutElastic 等 30+ 种） |
| `SetLoops(int, LoopType)` | 设置循环次数与类型（Restart / Yoyo / Incremental） |
| `SetAutoKill(bool)` | 完成后自动销毁（默认 `true`，循环动画需设为 `false`） |
| `SetId(object)` | 设置动画 ID，便于批量查找与管理 |
| `SetUpdate(bool)` | `true` 时不受 `Time.timeScale` 影响 |

### 回调与控制

**生命周期回调**：

```csharp
transform.DOMove(target, 1f)
    .OnStart(() => Debug.Log("动画开始"))
    .OnPlay(() => Debug.Log("播放"))
    .OnUpdate(() => Debug.Log("每帧"))
    .OnComplete(() => Debug.Log("完成"))
    .OnKill(() => Debug.Log("被销毁"))
    .OnRewind(() => Debug.Log("回退"))
    .OnPause(() => Debug.Log("暂停"));
```

**动画控制**：

```csharp
Tween tween = transform.DOMove(target, 1f).Pause(); // 创建后立即暂停

tween.Play();           // 恢复播放
tween.Pause();          // 暂停
tween.Kill();           // 销毁动画
tween.Restart();        // 重新开始
tween.DoPlayForward();  // 正向播放
tween.DoPlayBackwards();// 反向播放
tween.DoRestart();      // 从起始重新播放
```

**Tween vs Tweener**：`Tween` 是所有动画的基类，`Tweener` 是特定数值动画类型（如 `DOMove`、`DOFloat`）。通常使用 `Tween` 即可满足大多数场景。

**ID 系统**：通过 `SetId` 实现动画分组管理：

```csharp
// 创建多个共享 ID 的动画
transform.DOMoveX(5, 1f).SetId("MoveGroup");
otherTransform.DORotate(new Vector3(0, 180, 0), 1f).SetId("MoveGroup");

// 批量控制
DOTween.Pause("MoveGroup");  // 暂停所有 ID 为 "MoveGroup" 的动画
DOTween.Play("MoveGroup");   // 恢复
DOTween.Kill("MoveGroup");   // 全部销毁
```

### Sequence 序列动画

`Sequence` 用于编排多个动画的顺序、并行与时间轴对齐：

```csharp
Sequence sequence = DOTween.Sequence();

// 顺序执行
sequence.Append(transform.DOMove(target.position, 1f).SetEase(Ease.Linear));
sequence.Append(transform.DORotate(new Vector3(0, 180, 0), 1f).SetEase(Ease.OutBounce));
sequence.Append(transform.DOScale(new Vector3(2, 2, 2), 1f).SetEase(Ease.OutElastic));

sequence.Play();
sequence.OnComplete(() => Debug.Log("序列动画完成"));
```

**Sequence 专用方法**：

| 方法 | 说明 |
|------|------|
| `Append(Tween)` | 将动画添加到序列末尾 |
| `AppendCallback(TweenCallback)` | 在序列当前位置插入回调 |
| `AppendInterval(float)` | 插入指定秒数的空闲间隔 |
| `Prepend(Tween)` | 将动画添加到序列开头 |
| `PrependCallback(TweenCallback)` | 在序列开头插入回调 |
| `PrependInterval(float)` | 在序列开头插入间隔 |
| `Join(Tween)` | 与当前动画**同时播放**（并行） |
| `Insert(float, Tween)` | 在指定时间点插入动画 |
| `InsertCallback(float, TweenCallback)` | 在指定时间点插入回调 |

`Join` 是实现复杂并行动画的关键 — 例如移动的同时旋转和缩放可以 `Join` 而非 `Append`。

## Luban 配置表工具

[Luban](https://github.com/focus-creative-games/luban) 是 focal-creative-games 开源的一款游戏配置表解决方案，专为大中型项目的数值策划表管理而设计。

**安装**：

1. Clone `luban_example` 仓库并复制 demo 配置到项目。
2. 通过 git URL 安装 Unity Package：

```
https://github.com/focus-creative-games/luban_unity.git
```

**加载配置**：使用 `Tables` 类并注入自定义加载器：

```csharp
// 创建 Tables 实例，传入加载回调
Tables tables = new Tables(LoadTable);

// 自定义加载逻辑 — 从 Resources 读取 JSON 数据
private JArray LoadTable(string tableName)
{
    TextAsset textAsset = Resources.Load<TextAsset>($"ResExcel/JsonData/{tableName}");
    return JArray.Parse(textAsset.text);
}
```

Luban 的核心特性是一次性加载多个表格（table），解析后的强类型 C# 对象可直接在代码中使用，避免了手动解析 JSON/XML 的烦琐。配置表变更后仅需重新运行 Luban 生成工具即可更新类型定义，实现了**配置与代码分离**。

## 游戏研发流程

了解游戏从立项到上市的全流程，有助于架构师在设计阶段做出合理的可扩展性决策。

### 开发阶段

| 阶段 | 目标 | 关键活动 |
|------|------|----------|
| **调查立项** | 验证市场可行性 | 市场与竞品分析、确立题材与玩法、成本预估、团队建设、开发周期预估 |
| **原型阶段** | 验证玩法可玩性 | 实现基础框架与核心玩法，参与者以策划和程序为主 |
| **Alpha 阶段** | 丰富血肉 | 3 设计（宏观设计、剧情设计、故事板与环境设计）+ 3 制作（3D 建模、动画、声音），策划/程序/美术/音效全员参与 |
| **Beta 阶段** | 测试稳定 | 功能测试与迭代修复，测试团队介入 |
| **发行阶段** | 制作 Demo 对接渠道 | 发行运营部门参与，渠道对接 |
| **上市/运营** | 维护迭代 | 根据用户反馈持续更新，多平台版本移植 |

### 团队组成

游戏开发需六大职能协同：

1. **策划（产品）**：文案（文化内核）、数值（数值体系）、系统（框架与玩法逻辑）、战斗（关卡与战斗流程）。
2. **美术表现**：2D（原画、UI、场景）、3D（场景、角色建模）、动作动画、特效（技能、场景）、音频（音乐/音效/语音）、技术美术（TA）。
3. **技术研发**：客户端（Gameplay、图形、优化）、服务器（架构设计优化）、运维、质量保障（QA）、项目管理（PM）。
4. **发行运营**：产品向（活动策划、版本管理、本地化）、工具向（数据分析、用户研究）、市场向（渠道分发、调研）、用户向（社区管理、客服）。

**技术栈概览**：C++/C# 语言基础 → 设计模式与计算机系统 → 图形学（DirectX/OpenGL/Vulkan、光线追踪、GPU 编程）→ 引擎架构 → AI → 网络与数据库管理。Gameplay、UI 与 Shader 是客户端开发的三大核心支柱。

## 对象池与协程

这两个机制是 Unity 面试的高频考点，也是实际开发中优化性能与组织逻辑的核心工具。

### 对象池（Object Pool）

对象池是一种空间换时间/减少 GC 的设计模式，**用于存放反复创建/销毁的对象**。典型场景：子弹、敌人、粒子特效、UI 列表项。

核心思路：预先实例化一批对象放入池中，需要时取出并激活，用完后重置状态并放回池中（而非 `Destroy`），避免频繁的 `Instantiate` 与 `Destroy` 产生的 GC Alloc 和 CPU 开销。

### 协程（Coroutine）

协程基于 C# 的 `IEnumerator`（迭代器）实现。`yield return` 将执行挂起到下一次满足条件时恢复。协程运行在 Unity 主线程上，**不是多线程**。

**生命周期**：协程在 `Update` 之后、`LateUpdate` 之前执行。`SetActive(false)` 会完全停止该 GameObject 上已启动的协程，但 `enabled = false` 不会影响协程。

**常用 YieldInstruction**：

| 指令 | 说明 |
|------|------|
| `yield return null` / `0` | 等待下一帧 Update 完成后执行 |
| `WaitForSeconds` | 等待指定秒数（受 `Time.timeScale` 影响） |
| `WaitForSecondsRealtime` | 不受 `timeScale` 影响的秒数等待 |
| `WaitForEndOfFrame` | 等待当前帧所有渲染完成 |
| `WaitForFixedUpdate` | 等待下一次 FixedUpdate 后执行 |
| `WaitUntil(Func<bool>)` | 等待条件成立 |
| `WaitWhile(Func<bool>)` | 等待条件不成立 |

### MonoBehaviour 生命周期

理解生命周期是正确使用协程与事件订阅的前提：

```
Awake → OnEnable → Start → Update → FixedUpdate → LateUpdate → OnGUI
  → OnDisable → OnDestroy
```

- **Awake**：脚本实例创建时调用（无论 enabled 状态），用于获取自身组件引用。
- **OnEnable**：对象激活或 enabled 变为 true 时调用，用于事件订阅。
- **Start**：第一次 Update 之前调用，依赖其他组件初始化的逻辑放这里。
- **Update**：每帧调用，处理常规输入与逻辑。
- **FixedUpdate**：固定物理时间间隔调用，用于 Rigidbody 操作。
- **LateUpdate**：Update 之后调用，常用于相机跟随（确保跟随目标已移动完毕）。
- **OnDisable**：对象禁用时调用，用于取消事件订阅。
- **OnDestroy**：对象销毁时调用，用于清理资源。

### material vs sharedMaterial

- `renderer.material`：每次访问都会**创建材质副本**（Instance），修改仅影响当前 `MeshRenderer`。适合运行时临时修改颜色/透明度。
- `renderer.sharedMaterial`：指向**共享材质实例**，多个 `MeshRenderer` 共享同一材质。修改会**影响所有使用该材质的对象**，并写回 Project 中的材质资源。不推荐直接修改 `sharedMaterial`，应获取副本后操作。

---

> 选择建议：小型项目基础分层即可胜任；中大型项目推荐 MVVM 或 ECS 架构 + 事件驱动 + UniTask 异步 + DOTween 动画 + Luban 配置表的组合。核心衡量标准始终是：新成员能否快速理解、功能扩展是否灵活、Bug 定位是否高效。

---

## 程序集定义与自定义 Package

### 程序集定义（Assembly Definition）

默认情况下，Unity 将所有游戏脚本编译到同一个 `Assembly-CSharp.dll` 中。这带来三个问题：

+- **增量编译慢**：修改一个脚本触发全部重新编译
+- **无访问控制**：所有脚本可以访问其他脚本的类型，不利于模块化
+- **全平台编译**：所有脚本针对所有平台编译，无法按平台分离

通过 `.asmdef` 文件，可将脚本按模块划分为独立的程序集：

```
Assets/
├─ Scripts/
│  ├─ Core/
│  │  └─ Game.Core.asmdef        // 核心程序集
│  ├─ Characters/
│  │  └─ Game.Characters.asmdef  // 角色程序集
│  └─ UI/
│     └─ Game.UI.asmdef          // UI 程序集
```

**创建方式**：在目标文件夹中右键 → **Create** → **Scripting** → **Assembly Definition**。

**程序集定义引用（`.asmref`）**：当子文件夹需要显式归属到某个已有程序集时使用。创建 `.asmref` 资源并设置其 `Assembly Definition` 属性指向目标 `.asmdef`。

### 程序集引用规则

在 `.asmdef` 的 Inspector 中配置引用关系：

+- **Assembly Definition References**：引用其他自定义程序集
+- **Define Constraints**：条件编译符号约束——仅当定义了指定符号时，该引用才生效
+- **Platforms**：按平台勾选，排除不需要的平台编译

Unity 的引用限制：

+- 自定义程序集**不能**引用预定义程序集（如 `Assembly-CSharp.dll`）
+- 预定义程序集只能使用**自动引用**（Auto Referenced）中的代码
+- **禁止循环引用**：A 引用 B 且 B 引用 A 会导致编译错误

### 自定义 Package 布局

Unity Package 是代码和资源的独立分发单元。标准布局：

```
<package-root>/
  ├── package.json              // 包清单（name、version、dependencies）
  ├── README.md
  ├── CHANGELOG.md
  ├── LICENSE.md
  ├── Editor/                   // 编辑器专用代码（仅在 Editor 中编译）
  │   ├── Unity.[Name].Editor.asmdef
  │   └── EditorExample.cs
  ├── Runtime/                  // 运行时代码
  │   ├── Unity.[Name].asmdef
  │   └── RuntimeExample.cs
  ├── Tests/
  │   ├── Editor/
  │   │   ├── Unity.[Name].Editor.Tests.asmdef
  │   │   └── EditorExampleTest.cs
  │   └── Runtime/
  │       ├── Unity.[Name].Tests.asmdef
  │       └── RuntimeExampleTest.cs
  └── Documentation~            // 文档目录（~ 后缀表示元数据目录）
       └── [Name].md
```

**`package.json` 清单示例**：

```json
{
  "name": "com.example.mypackage",
  "version": "1.2.3",
  "displayName": "My Package",
  "description": "A custom Unity package",
  "unity": "2021.3",
  "dependencies": {
    "com.unity.textmeshpro": "3.0.0"
  },
  "keywords": ["example", "utility"],
  "author": {
    "name": "Author Name",
    "email": "author@example.com"
  }
}
```

> [!warning] 包名约束
> `name` 字段必须**全部小写字母**，使用反向域名格式（如 `com.company.package`），否则 Package Manager 会报错。

### 与项目架构的集成

程序集和 Package 在项目架构中的定位：

| 机制 | 粒度 | 适用场景 |
|:-----|:-----|:---------|
| `.asmdef` | 项目内部模块划分 | 隔离 Core/UI/Gameplay 等模块编译 |
| `.asmref` | 子文件夹归属 | 将零散脚本归入已有程序集 |
| Custom Package | 跨项目复用 | 通用工具库、框架、编辑器扩展 |
| UPM Package | 外部依赖管理 | 通过 git URL 或 OpenUPM 引入第三方库 |

> [!tip] 编译时间优化
> 合理使用 `.asmdef` 可将增量编译时间从几十秒降低到几秒。核心原则：**将不常变动的基础代码与频繁修改的业务代码分离到不同程序集**。
