---
title: "C# 多线程与并发编程 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [csharp, threading, task, async, concurrency, parallel]
source: "raw/cs/languages/csharp-threading.md"
---

# C# 多线程与并发编程 — 摘要

来源：`raw/cs/languages/csharp-threading.md`

## 概述

C# 并发模型演进全景：Thread → Task → async/await → Channel，涵盖废弃 API 的现代替代方案和线程安全三层防御。

## 要点

- **废弃 API**：`Thread.Suspend()`/`Resume()`/`Abort()`/`Interrupt()` 已在 .NET Core 中移除，抛出 `PlatformNotSupportedException`。现代 .NET 采用**协作式取消**（`CancellationToken`）
- **Thread 替代方案**：`ManualResetEventSlim`（暂停/恢复，用户态优先）、`CancellationToken`（停止信号）
- **Task**：所有现代并发工作的首选，由线程池管理。`Task.Factory.StartNew` + `TaskCreationOptions.LongRunning` 用于长期后台任务
- **async/await**：I/O 密集型核心，铁律 — 返回 `Task`/`Task<T>`、方法名以 `Async` 结尾、传递 `CancellationToken`、库代码用 `ConfigureAwait(false)`
- **`ValueTask<T>`**：高频短生命周期场景，同步完成时零分配
- **Channel**：生产者-消费者解耦，I/O 生产与 CPU 消费完全分离
- **线程安全三层**：`Interlocked`（原子操作）→ `lock`（Monitor）→ `Concurrent Collections`（无锁/低锁）；读多写少用 `ReaderWriterLockSlim`
- **决策树**：I/O → `async/await`，CPU → `Task.Run`/`Parallel`/`PLINQ`，流 → `Channel`，精细线程控制 → `Thread` + `CancellationToken`

## 关联页面

- [[concepts/CSharp并发模型|C# 并发模型]] — 概念综合页
- [[csharp-async-awaiter-摘要|C# 异步模型]] — TAP/async-await/Process 管理
