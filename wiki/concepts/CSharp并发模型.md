---
title: "C# 并发模型"
type: concept
updated: 2026-06-02
tags: [csharp, threading, task, async, concurrency, parallel]
aliases: [CSharp并发模型, C#多线程, .NET并发]
---

# C# 并发模型

C# 并发编程的演进路径：`Thread`（手动）→ `Task`（线程池抽象）→ `async/await`（I/O 密集）→ `Channel`（流式解耦）。核心原则：**协作式取消替代暴力终止**。

## 演进路线

```
Thread (手动控制)
  → Task (线程池抽象，现代默认选择)
    → async/await (I/O 密集型，不占线程等待)
      → Channel (生产者-消费者解耦)
```

## 废弃 API 与现代替代

.NET Core / .NET 5+ 中以下 API 已移除，调用抛 `PlatformNotSupportedException`：

| 废弃 API | 替代方案 |
|----------|---------|
| `Thread.Suspend()` / `Resume()` | `ManualResetEventSlim` |
| `Thread.Abort()` / `Interrupt()` | `CancellationToken` |

核心哲学：**协作式取消** — 线程自己主动检查信号并优雅退出，而非被外部暴力中断。

## Task — 现代并发的默认选择

`Task` 由线程池管理。`Task.Run` 将工作排队到线程池。长期后台任务（消息循环、持续监听）用 `Task.Factory.StartNew` + `TaskCreationOptions.LongRunning` 创建专用线程，避免长期占用线程池。

## async/await 铁律

| 应当 | 不应当 |
|------|--------|
| 返回 `Task` / `Task<T>` / `ValueTask<T>` | 不返回 `async void`（仅事件处理器例外） |
| 方法名 `Async` 结尾 | 不用 `Task.Wait()` / `Task.Result` 阻塞 |
| 传递 `CancellationToken` | 不在 `lock` 内 `await` |
| 库代码 `ConfigureAwait(false)` | 不在 ASP.NET Core 随意 `Task.Run` 外包 |

`ConfigureAwait(false)` 不捕获当前 `SynchronizationContext`，使 `await` 后续代码在线程池线程上执行。库代码用它避免死锁；UI 代码不应使用（需要回到 UI 线程）。

`ValueTask<T>`：高频短生命周期场景优先，同步完成时零分配。

## 线程安全三层防御

| 层级 | 机制 | 场景 |
|------|------|------|
| 1 | `Interlocked`（原子操作） | 简单计数器、标志位 |
| 2 | `lock`（Monitor） | 一般共享状态 |
| 3 | `Concurrent Collections`（无锁/低锁） | 高并发集合操作 |

`ReaderWriterLockSlim`：读多写少场景优于 `lock`，允许多读线程并行。

## Channel — 生产者-消费者解耦

```csharp
var channel = Channel.CreateBounded<byte[]>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// 生产者（I/O）→ channel.Writer
// 消费者（CPU）→ channel.Reader.ReadAllAsync()
```

I/O 密集型生产和 CPU 密集型消费完全分离，天然支持背压（BoundedChannelFullMode）。

## 决策指南

```
I/O 密集型（网络/文件/数据库）
  → async/await + Task/ValueTask，始终传递 CancellationToken

CPU 密集型（计算/处理）
  ├── 单次后台 → Task.Run
  ├── 批量并行 → Parallel.ForEachAsync / PLINQ
  └── 流式处理 → Channel

精细线程控制
  → Thread + CancellationToken + ManualResetEventSlim
```

## 关联页面

- [[sources/csharp-threading-摘要|C# 多线程来源摘要]]
- [[CSharp异步模型]] — async/await/TAP/ValueTask 深度解析
- [[CSharp进程管理]] — Process 异步等待与 CancellationToken
