---
title: "CSharp异步模型"
tags:
  - csharp
  - async
  - task
  - concurrency
created: 2026-06-02
---

# CSharp异步模型

C# 异步编程的完整知识体系，基于 TAP（Task-based Asynchronous Pattern），覆盖从 `async/await` 语法糖到底层状态机原理。

## 核心概念

- **TAP**：现代 .NET 异步编程的唯一推荐模式，使用 `async/await` + `Task<T>`
- **状态机**：编译器将 async 方法展开为 `IAsyncStateMachine`，每个 `await` 是状态切换点
- **Awaiter 模式**：`await` 不限于 Task，任何有 `GetAwaiter()` 的类型都可被 await
- **SynchronizationContext**：控制 await 后代码回到哪个线程；UI 框架利用它回到主线程

## 关键决策

| 场景 | 方案 |
|------|------|
| I/O 绑定（网络、数据库、文件） | 纯 `async/await`，不用 `Task.Run` |
| CPU 绑定（复杂计算） | `await Task.Run(() => Compute())` |
| 同步等待异步结果 | `GetAwaiter().GetResult()` 优于 `.Result` |
| 库代码 | 所有 await 处用 `ConfigureAwait(false)` |
| 事件处理 | `async void`（唯一合法场景） |

## 与已有页面关联

- [[CSharp并发模型]] — Task/Thread/ThreadPool 的并发原语对比
- [[CSharp内存GC]] — ValueTask 减少堆分配的原理与使用
- [[CSharp进程管理]] — Process 的异步等待与 CancellationToken
- [[CSharp网络Socket]] — Socket 的异步接收（ReceiveAsync/TAP）
- [[依赖注入]] — DI 容器中服务的异步工厂与生命周期
- [[concepts/Unity脚本架构|Unity 脚本架构]] — UniTask 在 Unity 中的异步实践与主线程安全

## 来源

- [[csharp-async-awaiter-摘要]]
