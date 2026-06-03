---
title: "CSharp异步模型"
tags:
  - csharp
  - async
  - task
  - concurrency
type: concept
updated: 2026-06-02
---

# CSharp异步模型

C# 异步编程的完整知识体系，基于 TAP（Task-based Asynchronous Pattern），覆盖从 `async/await` 语法糖到底层状态机原理，以及 IProgress、IAsyncEnumerable、Process 管理等进阶模式。

## 核心概念

- **TAP**：现代 .NET 异步编程的唯一推荐模式，使用 `async/await` + `Task<T>`
- **状态机**：编译器将 async 方法展开为 `IAsyncStateMachine`，每个 `await` 是状态切换点
- **Awaiter 模式**：`await` 不限于 Task，任何有 `GetAwaiter()` 的类型都可被 await
- **SynchronizationContext**：控制 await 后代码回到哪个线程；UI 框架利用它回到主线程
- **`IProgress<T>`**：TAP 进度报告接口，`Progress<T>` 自动捕获 SynchronizationContext 回到 UI 线程
- **`IAsyncEnumerable<T>`**（C# 8.0+）：`await foreach` 消费异步数据流，支持 `yield return` + `await`
- **PoolingAsyncValueTaskMethodBuilder**（.NET 6+）：状态机从对象池租用，减少 ValueTask 异步方法的分配
- **Process 进程管理**：`Process.Start` + `WaitForExitAsync(CancellationToken)` + 并发读取 stdout/stderr 防死锁

## 关键决策

| 场景 | 方案 |
|------|------|
| I/O 绑定（网络、数据库、文件） | 纯 `async/await`，不用 `Task.Run` |
| CPU 绑定（复杂计算） | `await Task.Run(() => Compute())` |
| 同步等待异步结果 | `GetAwaiter().GetResult()` 优于 `.Result` |
| 库代码 | 所有 await 处用 `ConfigureAwait(false)` |
| 事件处理 | `async void`（唯一合法场景） |
| 进度报告 | `IProgress<T>` + `Progress<T>` 自动回到 UI 线程 |
| 异步数据流 | `IAsyncEnumerable<T>` + `await foreach` |
| 进程管理 | `WaitForExitAsync(token)` + 并发读取 stdout/stderr |
| 高吞吐 ValueTask | `[AsyncMethodBuilder(typeof(PoolingAsyncValueTaskMethodBuilder))]` |

## `IAsyncEnumerable<T>` 示例

```csharp
// 生产者
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

// 消费者
await foreach (var line in ReadLinesAsync("large.log", token))
{
    Console.WriteLine(line);
}
```

## Process 异步管理示例

```csharp
using var process = new Process { StartInfo = startInfo };
process.Start();

// 必须并发读取 stdout/stderr，防止缓冲区满死锁
var outputTask = process.StandardOutput.ReadToEndAsync();
var errorTask = process.StandardError.ReadToEndAsync();

using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(2));
try
{
    await process.WaitForExitAsync(cts.Token);
}
catch (OperationCanceledException)
{
    process.Kill(entireProcessTree: true);
    throw new TimeoutException("进程执行超时");
}
```

## 与已有页面关联

- [[CSharp并发模型]] — Task/Thread/ThreadPool 的并发原语对比
- [[CSharp内存GC]] — ValueTask 减少堆分配的原理与使用
- [[CSharp进程管理]] — Process 的异步等待与 CancellationToken
- [[CSharp网络Socket]] — Socket 的异步接收（ReceiveAsync/TAP）
- [[依赖注入]] — DI 容器中服务的异步工厂与生命周期
- [[C++并发与异步]] — C++ `std::async`/`future`/`promise` 与 TAP 的对比
- [[comparisons/CSharp-vs-CPP-async|C# vs C++ 异步模型对比]] — 两种异步范式的系统对比分析
- [[concepts/Unity脚本架构|Unity 脚本架构]] — UniTask 在 Unity 中的异步实践与主线程安全
## 来源

- [[csharp-async-awaiter-摘要]]
- [[concepts/面试常见问题|面试常见问题 — C#]] — C# 技术面试 Q&A（装箱/GC/多态/多线程/异步/闭包/设计模式）