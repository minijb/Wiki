---
title: "C# 异步模型深度解析"
tags:
  - csharp
  - async
  - task
  - awaiter
  - statemachine
type: source
updated: 2026-06-02
source_files:
  - drafts/My_Vault/02_Knowledge/异步+多线程/
  - drafts/My_Vault/02_Knowledge/01_Language/Csharp_tools/Thread/
  - drafts/My_Vault/02_Knowledge/01_Language/Csharp_tools/Process/
  - drafts/My_Vault/02_Knowledge/01_Language/常用工具/00_CSharp_进程.md
---

## 0. 演进全景

C# 并发/异步编程的演进路径：

```
Thread (手动控制, .NET 1.0)
  → ThreadPool (线程复用, .NET 1.0)
    → Task (线程池抽象, .NET 4.0)
      → async/await (语言级异步, C# 5.0)
        → ValueTask (零分配异步, C# 7.0)
          → IAsyncEnumerable<T> (异步流, C# 8.0)
            → Channel<T> (生产者-消费者管道, .NET Core 3.0)
```

核心哲学：**协作式取消替代暴力终止**；**I/O 异步不占用线程**；**库代码用 ConfigureAwait(false)**。

# C# 异步模型深度解析

## 1. .NET 异步编程三大模式

.NET 提供了三种异步操作模式：

| 模式 | 全称 | 特征 | 现状 |
|------|------|------|------|
| **TAP** | Task-based Asynchronous Pattern | `async/await` + `Task<T>`，单个方法表示启动+完成 | **推荐** |
| **EAP** | Event-based Asynchronous Pattern | `Async` 后缀方法 + 事件/EventArg | 遗留 |
| **APM** | Asynchronous Programming Model | `BeginXxx`/`EndXxx` + `IAsyncResult` | **不推荐新开发** |

```csharp
// TAP —— 现代写法
public async Task<string> DownloadAsync(string url)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
}

// APM —— 旧式写法（应避免新代码使用）
public IAsyncResult BeginDownload(string url, AsyncCallback callback, object state)
{
    // ... BeginXxx 实现
}
```

---

## 2. Task 与 Task<T> 核心机制

### 2.1 任务状态机

`Task` 类的生命周期由 `TaskStatus` 枚举表示：

```
Created → WaitingForActivation → WaitingToRun → Running
                                                    ↓
                                    RanToCompletion / Canceled / Faulted
```

关键点：
- `new Task(...)` 创建的是**冷任务**（Cold Task），状态为 `Created`，不自动执行，需调用 `Start()`
- TAP 方法返回的任务必须是**热任务**（Hot Task），即已经启动的
- 如果 TAP 方法内部用 `new Task()` 构造返回，必须在返回前调用 `Start()`，否则调用方拿到一个永远不会完成的任务

```csharp
// 错误：返回冷任务
Task<int> BadAsync()
{
    var t = new Task<int>(() => 42);  // Created 状态
    return t;  // 调用方 await 会永久挂起！
}

// 正确：启动后再返回，或用 Task.Run
Task<int> GoodAsync()
{
    var t = new Task<int>(() => 42);
    t.Start();
    return t;
}
// 更推荐：
Task<int> BetterAsync() => Task.Run(() => 42);
```

### 2.2 ValueTask —— 减少堆分配

`Task` 是引用类型，每次返回都会在堆上分配。对于高频调用或同步完成的路径，`ValueTask<T>` 作为值类型可以消除分配：

```csharp
// 场景：缓存命中时同步返回，避免分配 Task 对象
private Dictionary<string, string> _cache = new();

public async ValueTask<string> GetValueAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return value;  // 同步返回，无堆分配

    value = await FetchFromDbAsync(key);
    _cache[key] = value;
    return value;
}
```

**注意事项**：
- `ValueTask` 只能被 await **一次**；多次 await 行为未定义
- 不要 `.Result` 或 `.Wait()` 一个 `ValueTask`
- 若需要多次等待或组合，用 `.AsTask()` 转为 `Task`

### 2.3 Task.WhenAll / WhenAny 组合器

```csharp
// 并行发起多个请求
var tasks = userIds.Select(id => GetUserAsync(id));
User[] users = await Task.WhenAll(tasks);

// 竞速：只取最快完成的
var fastest = await Task.WhenAny(tasks);
```

**注意**：LINQ 的延迟执行与异步混合时的陷阱 —— `Select` 返回的 `IEnumerable` 在迭代前不会真正发起异步调用。

```csharp
// 错误 —— GetUserAsync 在 ToArray() 前不会被调用
var getUserTasks = userIds.Select(id => GetUserAsync(id));  // 仍是 IEnumerable
return await Task.WhenAll(getUserTasks);  // 这里才逐个调用

// 正确 —— 提前物化
var getUserTasks = userIds.Select(id => GetUserAsync(id)).ToArray();
return await Task.WhenAll(getUserTasks);
```

---

## 3. async/await 编译器变换 —— 状态机原理

### 3.1 链式操作等价形式

`async/await` 的底层等价于 `ContinueWith` 链式调用：

```csharp
// async/await 版本
public async Task<int> ComputeAsync()
{
    int a = await FetchAAsync();
    int b = await FetchBAsync();
    return a + b;
}

// 等效的 ContinueWith 版本
public Task<int> ComputeWithContinuation()
{
    return FetchAAsync().ContinueWith(tA =>
        FetchBAsync().ContinueWith(tB =>
            tA.Result + tB.Result
        )
    ).Unwrap();
}
```

### 3.2 状态机展开

编译器将 `async` 方法转换为一个实现了 `IAsyncStateMachine` 的结构体：

```csharp
// 原始代码
public static async Task TestAsync()
{
    Console.WriteLine("步骤1");
    await WorkAsync();
    Console.WriteLine("步骤2");
    await WorkAsync();
    Console.WriteLine("步骤3");
}

// 编译器生成的状态机（简化）
public class TestAsyncStateMachine : IAsyncStateMachine
{
    public int _state = 0;     // 状态：0→步骤1之后，1→步骤2之后，-1→结束
    private TaskAwaiter _awaiter;

    public void MoveNext()
    {
        switch (_state)
        {
            case 0:
                Console.WriteLine("步骤1");
                _awaiter = WorkAsync().GetAwaiter();
                if (_awaiter.IsCompleted) goto case 1;
                _state = 1;
                _awaiter.OnCompleted(MoveNext);  // 注册回调
                return;
            case 1:
                _awaiter.GetResult();  // 获取结果（如有）
                Console.WriteLine("步骤2");
                _awaiter = WorkAsync().GetAwaiter();
                if (_awaiter.IsCompleted) goto case -1;
                _state = -1;
                _awaiter.OnCompleted(MoveNext);
                return;
            default:
                Console.WriteLine("步骤3");
                return;
        }
    }
}
```

### 3.3 关键结论

- `async` 关键字**只用于允许方法内使用 `await`**，本身不改变返回值类型
- `await` 会挂起当前方法，释放线程回调用方，等任务完成后从挂起点继续
- **async 不代表多线程**：异步可以在单线程上通过时间片轮转实现
- 如果一个 `async` 方法中没有 `await`，编译器只给警告，代码仍会执行，但状态机是空转

---

## 4. Awaiter 模式

### 4.1 什么是 Awaiter

`await` 并不限于 `Task`。编译器要求被 await 的类型满足：
1. 有 `GetAwaiter()` 方法（可以是扩展方法）
2. 返回的 Awaiter 实现 `INotifyCompletion` 接口
3. Awaiter 有 `IsCompleted` 属性
4. Awaiter 有 `GetResult()` 方法

```csharp
public interface INotifyCompletion
{
    void OnCompleted(Action continuation);
}
```

### 4.2 自定义 Awaiter

```csharp
// 将 await 后的代码切换到线程池执行的自定义 Awaiter
public struct SwitchToThreadPoolAwaiter : INotifyCompletion
{
    public bool IsCompleted => false;  // 始终异步

    public void GetResult() { }        // 无返回值版本

    public void OnCompleted(Action continuation)
    {
        ThreadPool.QueueUserWorkItem(_ => continuation());
    }

    public SwitchToThreadPoolAwaiter GetAwaiter() => this;
}

// 使用
public static async Task Example()
{
    Console.WriteLine($"当前线程池线程？{Thread.CurrentThread.IsThreadPoolThread}"); // false
    await new SwitchToThreadPoolAwaiter();
    Console.WriteLine($"当前线程池线程？{Thread.CurrentThread.IsThreadPoolThread}"); // true
}
```

### 4.3 TaskAwaiter 与 GetResult

`Task.GetAwaiter().GetResult()` 是同步阻塞等待的**推荐方式**，优于 `.Result` 和 `.Wait()`：

| 方式 | 异常包装 | 死锁风险 |
|------|----------|----------|
| `task.Result` | 抛 `AggregateException` 包装原始异常 | 高 |
| `task.Wait()` | 抛 `AggregateException` | 高 |
| `task.GetAwaiter().GetResult()` | 直接抛原始异常 | 中（仍会阻塞） |

---

## 5. 同步上下文与死锁

### 5.1 SynchronizationContext

在 UI 线程（WPF/WinForms）或 ASP.NET（旧版）中，`await` 默认会捕获当前 `SynchronizationContext`，并在任务完成后回到原始上下文继续执行：

```csharp
// UI 事件处理中 —— 自动回到 UI 线程更新控件
private async void Button_Click(object sender, EventArgs e)
{
    // 在 UI 线程
    var data = await FetchDataAsync();  // 异步操作在后台执行
    label.Text = data;  // 自动回到 UI 线程更新
}
```

### 5.2 ConfigureAwait(false)

```csharp
// 库代码中：避免回到原始上下文，减少开销
public async Task<string> LibraryMethodAsync()
{
    var data = await FetchAsync().ConfigureAwait(false);
    // 这里可能在任意线程，不能访问 UI 控件
    return ProcessData(data);
}
```

- `ConfigureAwait(true)`（默认）：回到原始上下文
- `ConfigureAwait(false)`：不回到原始上下文，在任意线程继续
- **推荐**：库代码统一使用 `ConfigureAwait(false)`；应用层顶层方法才回到原始上下文

### 5.3 经典死锁场景

```csharp
// 死锁示例 —— 在 UI 线程同步等待异步方法
public void Button_Click()
{
    var result = FetchAsync().Result;  // 死锁！
    // .Result 阻塞 UI 线程
    // FetchAsync 内部 await 后想回到 UI 线程继续
    // UI 线程被 .Result 阻塞 → 永远回不来
}

// 修复方案一：async all the way
private async void Button_Click()
{
    var result = await FetchAsync();
}

// 修复方案二：ConfigureAwait(false) 避免回到 UI 线程
// 在 FetchAsync 内部所有 await 处添加 .ConfigureAwait(false)
```

---

## 6. 多线程基础

### 6.1 Thread 创建方式

```csharp
// 方式1：Thread
var t = new Thread(() => Console.WriteLine("Worker"));
t.IsBackground = true;  // 后台线程：主线程退出时自动终止
t.Priority = ThreadPriority.Normal;
t.Start();

// 方式2：ThreadPool
ThreadPool.QueueUserWorkItem(_ => DoWork());

// 方式3：Task（推荐）
Task.Run(() => DoWork());
```

### 6.2 Task 创建选项

```csharp
// Task.Run —— 在线程池上排队
Task.Run(() => CpuWork());

// Task.Factory.StartNew —— 更多控制
Task.Factory.StartNew(() => LongWork(),
    TaskCreationOptions.LongRunning);  // 提示调度器：应使用专用线程

// Task.Delay —— 非阻塞延迟（优于 Thread.Sleep）
await Task.Delay(1000);  // 异步，不阻塞线程
```

### 6.3 线程同步原语

| 类型 | 描述 | 异步可用 |
|------|------|----------|
| `lock` / `Monitor` | 排他锁 | ❌（不能在 await 中使用） |
| `Mutex` | 跨进程互斥锁 | ❌ |
| `Semaphore` / `SemaphoreSlim` | 信号量，控制并发数 | SemaphoreSlim ✅ |
| `ReaderWriterLockSlim` | 读写锁 | ❌ |
| `ManualResetEvent` / `AutoResetEvent` | 事件通知 | Slim 版本有限支持 |
| `Interlocked` | 原子操作 | ✅（CPU 操作，不阻塞） |
| `Channel<T>` | 生产者-消费者管道 | ✅ |
| `ConcurrentDictionary/Queue/Bag` | 线程安全集合 | ✅ |

### 6.4 Parallel 与 PLINQ

```csharp
// Parallel.For —— 数据并行
Parallel.For(0, 100, i => Process(i));

// Parallel.ForEach
Parallel.ForEach(items, item => Process(item));

// Parallel.Invoke —— 任务并行
Parallel.Invoke(
    () => TaskA(),
    () => TaskB(),
    () => TaskC()
);

// PLINQ —— 并行 LINQ
var results = items.AsParallel()
                   .WithDegreeOfParallelism(4)
                   .Where(x => ExpensiveFilter(x))
                   .Select(x => Transform(x))
                   .AsOrdered()    // 保留顺序
                   .ToArray();
```

---

## 7. CancellationToken 取消机制

### 7.1 基本用法

```csharp
// 创建取消令牌源
var cts = new CancellationTokenSource();
CancellationToken token = cts.Token;

// 传递 token 给异步操作
try
{
    await Task.Delay(5000, token);
    // 或
    await SomeOperationAsync(token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("操作已取消");
}
finally
{
    cts.Dispose();  // CancellationTokenSource 实现了 IDisposable
}
```

### 7.2 超时取消

```csharp
// 三种超时方式
var cts1 = new CancellationTokenSource(3000);          // 3秒后自动取消
var cts2 = new CancellationTokenSource(TimeSpan.FromSeconds(3));
cts.CancelAfter(3000);                                // 手动设置
```

### 7.3 取消检测与善后

```csharp
// 主动检查
token.IsCancellationRequested   // 查询是否已请求取消
token.ThrowIfCancellationRequested()  // 已取消则抛异常

// 注册取消回调（善后清理）
token.Register(() => Console.WriteLine("清理资源..."));

// 前注册后执行：多个 Register 的回调按注册的逆序执行

// 提前返回一个 Cancelled 的 Task（不抛异常）
public Task<string> GetDataAsync(CancellationToken token)
{
    if (token.IsCancellationRequested)
        return Task.FromCanceled<string>(token);
    // ...
}
```

### 7.4 推荐方法签名

```csharp
// 重载模式：默认 CancellationToken.None
async Task FooAsync(int delay, CancellationToken token = default)
{
    await Task.Delay(delay, token);
}

// 或使用 Nullable
async Task FooAsync(int delay, CancellationToken? token = null)
{
    var ct = token ?? CancellationToken.None;
    await Task.Delay(delay, ct);
}
```

---

## 8. async void 陷阱

```csharp
// async void —— 仅用于事件处理
button.Click += async (s, e) => await OnClickAsync();

// 问题：
// 1. 异常无法被外部捕获 —— 直接抛到 SynchronizationContext，可能崩溃进程
// 2. 调用方无法等待完成
// 3. 难以单元测试

// 所有非事件处理的异步方法必须返回 Task 或 Task<T>
```

---

## 9. TAP 进度报告：IProgress\<T\>

TAP 通过 `IProgress<T>` 接口实现异步操作的进度通知，避免回调耦合：

```csharp
public interface IProgress<in T>
{
    void Report(T value);
}
```

默认实现 `Progress<T>` 会自动捕获 `SynchronizationContext`，将 `Report` 回调调度到创建它的线程（通常是 UI 线程）：

```csharp
public async Task ProcessFilesAsync(
    string[] files, IProgress<int>? progress = null)
{
    for (int i = 0; i < files.Length; i++)
    {
        await ProcessFileAsync(files[i]);
        progress?.Report((i + 1) * 100 / files.Length);  // 百分比
    }
}

// UI 调用
var progress = new Progress<int>(pct =>
    progressBar.Value = pct);  // 自动回到 UI 线程更新
await ProcessFilesAsync(files, progress);
```

---

## 10. IAsyncEnumerable\<T\> 与异步流 (C# 8.0+)

`IAsyncEnumerable<T>` 支持 `await foreach` 语法，将异步数据流以迭代器方式消费：

```csharp
// 生产者：yield return 配合 await
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken token = default)
{
    using var reader = new StreamReader(path);
    while (!reader.EndOfStream)
    {
        token.ThrowIfCancellationRequested();
        yield return await reader.ReadLineAsync();
    }
}

// 消费者：await foreach
await foreach (var line in ReadLinesAsync("large.log", token))
{
    Console.WriteLine(line);
}

// 配合 System.Linq.Async 进行 LINQ 操作
await foreach (var item in source
    .WhereAwait(async x => await FilterAsync(x))
    .SelectAwait(async x => await TransformAsync(x)))
{
    Process(item);
}
```

> [!note] CancellationToken with IAsyncEnumerable
> 使用 `[EnumeratorCancellation]` 特性标记 token 参数，编译器自动将其绑定到 `await foreach` 的 `WithCancellation()` 调用。

---

## 11. ValueTask 池化 (PoolingAsyncValueTaskMethodBuilder)

.NET 6+ 引入了 `PoolingAsyncValueTaskMethodBuilder`，`ValueTask` 异步方法的状态机可从对象池租用，进一步减少分配：

```csharp
// 标记为使用池化状态机
[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]
public async ValueTask<int> GetValueAsync()
{
    await Task.Delay(100);
    return 42;
}

// 或通过项目文件全局启用
// <PoolingAsyncValueTaskMethodBuilder>true</PoolingAsyncValueTaskMethodBuilder>
```

适用条件：
- 方法返回 `ValueTask` 或 `ValueTask<T>`
- 方法不会在多个线程上并发执行 `MoveNext`
- 高频调用场景收益最大

---

## 12. Process 进程管理

`System.Diagnostics.Process` 用于启动和管理外部进程，实现 `IDisposable`：

### 12.1 两种启动方式

```csharp
// 方式1：静态便捷方法
Process.Start("notepad.exe");
Process.Start("notepad.exe", "file.txt");

// 方式2：ProcessStartInfo 精细控制（推荐）
using var process = new Process
{
    StartInfo = new ProcessStartInfo
    {
        FileName = "python",
        Arguments = "script.py --verbose",
        UseShellExecute = false,          // 重定向流必须 false
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        CreateNoWindow = true,
        WorkingDirectory = "/path/to/work"
    }
};
process.Start();
```

### 12.2 ProcessStartInfo 关键属性

| 属性 | 说明 |
|------|------|
| `FileName` | 可执行文件路径或文档 |
| `Arguments` | 命令行参数 |
| `UseShellExecute` | 是否用 OS Shell 启动；**与 `RedirectStandard*` 互斥** |
| `CreateNoWindow` | 不显示新窗口（仅 Windows） |
| `WorkingDirectory` | 工作目录 |
| `RedirectStandardOutput/Error/Input` | 重定向标准流 |
| `StandardOutputEncoding/ErrorEncoding` | 流编码 |
| `Environment` | 环境变量字典 |

### 12.3 异步等待与超时

```csharp
using var process = new Process { StartInfo = startInfo };
process.Start();

// 并发读取 stdout/stderr（防止缓冲区满死锁）
var outputTask = process.StandardOutput.ReadToEndAsync();
var errorTask = process.StandardError.ReadToEndAsync();

// 带超时的异步等待
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(2));
try
{
    await process.WaitForExitAsync(cts.Token);
}
catch (OperationCanceledException)
{
    process.Kill(entireProcessTree: true);  // .NET 6+
    throw new TimeoutException("进程执行超时");
}

var output = await outputTask;
var error = await errorTask;
```

> [!warning] Token 取消不会自动杀进程
> `CancellationToken` 被取消只停止等待，**必须显式调用 `process.Kill()`**。

### 12.4 输出重定向死锁

stdout 和 stderr 各有约 4KB 缓冲区。**必须并发读取**，否则子进程写满一个缓冲区后阻塞，父进程在等待另一个流 → 死锁。

```csharp
// ❌ 错误：顺序读取
var output = process.StandardOutput.ReadToEndAsync();
var error = process.StandardError.ReadToEndAsync();  // 可能永远等不到！
await process.WaitForExitAsync();

// ✅ 正确：并发读取
var outputTask = process.StandardOutput.ReadToEndAsync();
var errorTask = process.StandardError.ReadToEndAsync();
await process.WaitForExitAsync(token);
```

### 12.5 运行 Python 脚本

```csharp
public static async Task<string> RunPythonAsync(
    string scriptPath, string args)
{
    var startInfo = new ProcessStartInfo
    {
        FileName = "python",
        Arguments = $"{scriptPath} {args}",
        UseShellExecute = false,
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        WorkingDirectory = Path.GetDirectoryName(scriptPath)
    };

    using var process = new Process { StartInfo = startInfo };
    process.Start();

    var outputTask = process.StandardOutput.ReadToEndAsync();
    var errorTask = process.StandardError.ReadToEndAsync();
    await process.WaitForExitAsync();

    if (process.ExitCode != 0)
        throw new InvalidOperationException(
            $"Python failed: {await errorTask}");

    return await outputTask;
}
```

---

## 13. I/O 绑定 vs CPU 绑定
| 维度 | I/O 绑定 | CPU 绑定 |
|------|----------|----------|
| 典型场景 | 网络请求、数据库查询、文件读写 | 复杂计算、图像处理 |
| 使用方式 | 纯 `async/await`（不用 `Task.Run`） | `await Task.Run(() => Compute())` |
| 线程 | 不占用线程（利用 IO 完成端口） | 使用线程池线程 |
| 关键原则 | 让操作系统异步 I/O 负责 | 把计算丢给线程池 |

```csharp
// I/O 绑定：不要用 Task.Run 包一层！
// 错误：
var data = await Task.Run(() => httpClient.GetStringAsync(url));
// 正确：
var data = await httpClient.GetStringAsync(url);

// CPU 绑定：用 Task.Run 释放 UI 线程
var result = await Task.Run(() => ComplexCalculation());
```

---

## 10. 常见面试要点

1. **async/await 原理**：编译器生成状态机（`IAsyncStateMachine`），await 点是状态切换点
2. **Task vs ValueTask**：Task 是引用类型（堆分配），ValueTask 是值类型（减少分配，但只能 await 一次）
3. **死锁场景**：UI 线程 `.Result` / `.Wait()` 同步等待 → 异步方法在 UI 线程上 await 后回不来
4. **SynchronizationContext**：await 默认捕获，UI 框架依赖它回到 UI 线程；库代码用 `ConfigureAwait(false)` 避免
5. **async void**：仅用于事件处理，否则异常无法捕获
6. **CancellationToken**：推荐所有异步方法都接受 token 参数；`CancellationTokenSource` 需要 Dispose
7. **锁不能跨 await**：`lock` 内部不能有 `await`；用 `SemaphoreSlim` 替代
8. **Task.Run 使用场景**：仅用于 CPU 绑定工作；I/O 绑定直接用原生异步方法
