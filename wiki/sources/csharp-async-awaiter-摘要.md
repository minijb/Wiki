---
title: "CSharp异步模型-摘要"
source: "raw/cs/languages/csharp-async-awaiter.md"
type: source-summary
created: 2026-06-02
---

# CSharp异步模型-摘要

> 源文件: [[CSharp异步模型]]

## 核心内容

深度解析 C# 异步编程的完整体系：.NET 三种异步模式（TAP/EAP/APM）、Task/ValueTask 机制、async/await 编译器状态机变换、自定义 Awaiter 模式、SynchronizationContext 与死锁场景、多线程同步原语对比、CancellationToken 取消机制、I/O 绑定 vs CPU 绑定的正确使用方式。

## 关键要点

1. **TAP 是唯一推荐模式**：`async/await` + `Task<T>`；APM（Begin/End）和 EAP 已过时
2. **状态机原理**：编译器将 async 方法转换为 `IAsyncStateMachine`，每个 await 是状态切换点
3. **Task vs ValueTask**：Task 堆分配，ValueTask 值类型减少分配但只能 await 一次
4. **死锁根因**：UI 线程同步等待（.Result/.Wait）→ await 后回不到被阻塞的线程
5. **ConfigureAwait(false)**：库代码避免回到原始上下文；应用层顶层保留默认行为
6. **CancellationToken**：推荐所有异步方法接受 token；CancellationTokenSource 需 Dispose
7. **async void 仅用于事件处理**：否则异常无法捕获，且调用方无法等待

## 相关页面

- [[CSharp并发模型]] — 已有 wiki 页面
- [[CSharp内存GC]]
- [[CSharp集合框架]]
