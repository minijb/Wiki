---
title: "C# 多线程与并发编程"
date: 2026-05-10
tags: [csharp, thread, task, async, concurrency, parallel]
type: language
aliases: [线程启动, 多线程, 异步编程]
description: C# 多线程/并发编程演进：Thread → Task → async/await → Channel，含废弃 API 的替代方案
---

# C# 多线程与并发编程

## 重要警告：已废弃的 API

以下 API 在 .NET Core / .NET 5+ 中**已完全移除**，调用会抛出 `PlatformNotSupportedException`：

| 废弃 API               | 移除原因                         | 替代方案                         |
| -------------------- | ---------------------------- | ---------------------------- |
| `Thread.Suspend()`   | 可能在持有锁时挂起，导致死锁               | `ManualResetEventSlim`       |
| `Thread.Resume()`    | 同上，且线程完全被动无法清理资源             | `ManualResetEventSlim.Set()` |
| `Thread.Abort()`     | 异步异常破坏状态一致性，`finally` 块可能不执行 | `CancellationToken`          |
| `Thread.Interrupt()` | 同样破坏控制流                      | `CancellationToken`          |

> [!warning] 核心原则
> 现代 .NET 采用**协作式取消**（Cooperative Cancellation）——不是从外部杀死线程，而是让线程自己主动检查信号并优雅退出。

以下各节从底层到高层展示现代替代方案，由 `Thread` 的手动控制逐步过渡到 `Task`/`async-await` 的高级抽象。

## 1. Thread：最基础的线程启动

```csharp
Thread t = new Thread(() =>
{
    Console.WriteLine("开始下载:" + Thread.CurrentThread.ManagedThreadId);
    Thread.Sleep(2000);
    Console.WriteLine("下载完成");
});
t.Start();
```

### 带 CancellationToken 的安全终止

```csharp
using var cts = new CancellationTokenSource();

var thread = new Thread(() =>
{
    try
    {
        while (!cts.Token.IsCancellationRequested)
        {
            DoWork();
            cts.Token.ThrowIfCancellationRequested();
        }
    }
    catch (OperationCanceledException)
    {
        Cleanup();  // 正常退出路径
    }
});

thread.Start();

// 安全终止
cts.Cancel();
thread.Join();  // 等待自然退出
```

### 暂停/恢复的替代方案：ManualResetEventSlim

```csharp
private readonly ManualResetEventSlim _pauseEvent = new(true);
private CancellationTokenSource _stopCts = new();

void ThreadProc()
{
    while (!_stopCts.Token.IsCancellationRequested)
    {
        _pauseEvent.Wait(_stopCts.Token);  // 信号态通过，取消时抛出
        DoWork();
    }
}

// 控制
_pauseEvent.Reset();     // 暂停
_pauseEvent.Set();       // 恢复
_stopCts.Cancel();       // 停止
_pauseEvent.Set();       // 唤醒后退出循环
```

> [!tip] 推荐做法
> `ManualResetEventSlim` 优先于 `ManualResetEvent` —— 用户态优先，减少内核切换开销。
> 现代代码中也可用 `CancellationTokenSource` 替代 `volatile bool` 作为停止信号——Token 注册回调机制比内存屏障更安全、语义更明确。

## 2. Task：现代并发的默认选择

`Task` 是 `Thread` 的高级抽象，由线程池管理，是**所有现代并发工作的首选**。

`Task.Run` 将工作排队到线程池，适合短期任务。对于长期运行的后台任务（如消息循环、持续监听），用 `Task.Factory.StartNew` 配合 `TaskCreationOptions.LongRunning` 创建**专用线程**，避免长期占用线程池线程：

```csharp
// 长期后台任务（消息循环、持续监听等）→ 专用线程，不阻塞线程池
var listener = Task.Factory.StartNew(() =>
{
    while (!ct.IsCancellationRequested)
    {
        ListenForMessages(ct);
    }
}, ct, TaskCreationOptions.LongRunning, TaskScheduler.Default);
```

```csharp
// Task.Run 启动
Task task = Task.Run(() =>
{
    Console.WriteLine($"开始下载, 线程ID:{Thread.CurrentThread.ManagedThreadId}");
    Thread.Sleep(500);
    Console.WriteLine("下载完成!");
});

await task;  // 非阻塞等待
```

### 带 CancellationToken 的 Task

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

var task = Task.Run(() =>
{
    for (int i = 0; i < 100; i++)
    {
        cts.Token.ThrowIfCancellationRequested();
        Thread.Sleep(100);
    }
}, cts.Token);

try
{
    await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("任务已取消");
}
```

## 3. async/await：I/O 密集型的核心

**原则：I/O 操作始终使用 `async/await`，不占用线程等待。**

```csharp
// ✅ 正确
public async Task<string> GetDataAsync(CancellationToken ct = default)
{
    var data = await httpClient.GetStringAsync(url, ct);
    return data;
}
```

### 铁律

| ✅ DO | ❌ DON'T |
|-------|----------|
| 返回 `Task` / `Task<T>` / `ValueTask<T>` | 不要返回 `async void`（仅事件处理器例外） |
| 方法名以 `Async` 结尾 | 不要用 `Task.Wait()` / `Task.Result` 阻塞 |
| 传递 `CancellationToken` | 不要在 `lock` 内 `await` |
| 库代码用 `ConfigureAwait(false)` | 不要在 ASP.NET Core 中随意用 `Task.Run` 外包 |

> [!note] ConfigureAwait(false) 原理
> 不捕获当前 `SynchronizationContext`（UI 线程/ASP.NET 请求上下文），使 `await` 之后的代码在线程池线程上继续执行。UI 代码需要回到 UI 线程，故不适用；库代码不关心上下文，应用它避免死锁和性能开销。

> [!note] "不要外包"的含义
> 在已经是异步的调用上再包装 `Task.Run(() => SomeAsyncMethod())` ——这多占用一个线程池线程去等待一个异步操作，纯粹浪费。`Task.Run` 应只用于将**同步 CPU 密集工作**移到后台线程。

### `ValueTask<T>`：高频短生命周期优先

```csharp
// 同步完成时零分配，异步时也轻于 Task<T>
public ValueTask<int> GetValueAsync()
{
    if (_cached) return new ValueTask<int>(_value);
    return new ValueTask<int>(FetchFromDbAsync());
}
```

### PeriodicTimer（.NET 6+）— 现代异步定时器

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
while (await timer.WaitForNextTickAsync(ct))
{
    DoPeriodicWork();
}
```

无需 `System.Timers.Timer` 的事件回调，天然支持异步 `await` 和 `CancellationToken`。

## 4. 线程池 ThreadPool

```csharp
ThreadPool.QueueUserWorkItem(state =>
{
    string[] arr = state as string[];
    Console.WriteLine($"处理: {arr[0]}, 线程ID: {Thread.CurrentThread.ManagedThreadId}");
}, new string[] { "test" });

// 获取线程池信息
ThreadPool.GetMaxThreads(out int workerThreads, out int completionPortThreads);
Console.WriteLine($"Worker: {workerThreads}, IOCP: {completionPortThreads}");
```

> [!tip]
> 现代代码中优先使用 `Task.Run`（底层使用线程池），而非直接调用 `ThreadPool.QueueUserWorkItem`。

## 5. CPU 密集型：Parallel / PLINQ

```csharp
// Parallel.ForEach
Parallel.ForEach(items, item => HeavyComputation(item));

// PLINQ
var results = items
    .AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .Select(x => Process(x))
    .ToList();

// Parallel.ForEachAsync (.NET 6+)
await Parallel.ForEachAsync(items, async (item, ct) =>
{
    await ProcessAsync(item, ct);
});
```

## 6. Channel：生产者-消费者解耦

将 I/O 密集型生产和 CPU 密集型消费完全分离：

```csharp
var channel = Channel.CreateBounded<byte[]>(new BoundedChannelOptions(100)
{
    SingleWriter = true,
    SingleReader = true,
    FullMode = BoundedChannelFullMode.Wait  // 默认：队列满时阻塞等待；也可选 DropNewest/DropOldest/DropWrite
});

// 生产者（I/O 密集）
// 注意：生产环境中应为 fire-and-forget Task 添加异常处理，避免未观察到的异常导致进程崩溃
_ = Task.Run(async () =>
{
    try
    {
        await foreach (var chunk in ReadFileChunksAsync("data.log"))
            await channel.Writer.WriteAsync(chunk);
        channel.Writer.Complete();
    }
    catch (Exception ex)
    {
        channel.Writer.Complete(ex);
    }
});

// 消费者（CPU 密集）
await foreach (var chunk in channel.Reader.ReadAllAsync())
{
    ProcessChunk(chunk);
}
```

## 7. 线程安全

三层防御策略：

```csharp
// 层级 1：Interlocked（最轻，原子操作）
Interlocked.Increment(ref _counter);

// 层级 2：lock（Monitor）
lock (_lockObj) { _sharedState.Update(); }

// 层级 3：Concurrent Collections（无锁/低锁）
var dict = new ConcurrentDictionary<string, int>();
var queue = new ConcurrentQueue<Item>();
var bag   = new ConcurrentBag<Item>();
```

| 集合 | 适用场景 |
|------|---------|
| `ConcurrentDictionary<TKey,T>` | 并发读写字典 |
| `ConcurrentQueue<T>` | 无锁 FIFO 队列 |
| `ConcurrentBag<T>` | 无序、局部线程优先 |
| `BlockingCollection<T>` | 有界生产者-消费者 |
| `Channel<T>` | 高性能异步管道 |

### ReaderWriterLockSlim — 读多写少的优化锁

```csharp
private readonly ReaderWriterLockSlim _rwLock = new();

// 读（多线程可同时进入）
_rwLock.EnterReadLock();
try { return _sharedData; }
finally { _rwLock.ExitReadLock(); }

// 写（独占）
_rwLock.EnterWriteLock();
try { _sharedData = newValue; }
finally { _rwLock.ExitWriteLock(); }
```

> [!tip] 读多写少场景
> 读远多于写时，`ReaderWriterLockSlim` 优于 `lock`——允许多个读线程并行，只有写操作互斥。

## 8. 决策指南

```
操作类型？
├── I/O 密集型（网络/文件/数据库）
│     → async/await + Task/ValueTask
│     → 始终传递 CancellationToken
│
├── CPU 密集型（计算/处理）
│     ├── 单次后台任务 → Task.Run
│     ├── 批量并行 → Parallel.ForEachAsync / PLINQ
│     └── 流式处理 → Channels / Pipelines
│
└── 需要精细控制线程生命周期
      → Thread + CancellationToken + ManualResetEventSlim
```

> [!note] 一句话总结
> I/O 用 `async/await`，CPU 用 `Task.Run`/`Parallel`/`PLINQ`，流用 `Channel`，永远不用 `Suspend`/`Resume`/`Abort`。
