---
title: "C++ 并发与异步编程 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [cpp, concurrency, thread, async, future]
source: "raw/cs/languages/cpp-concurrency.md"
---

# C++ 并发与异步编程 — 摘要

来源：`raw/cs/languages/cpp-concurrency.md`

## 概述

C++ 并发编程完整指南，从底层 `std::thread` 创建与管理，到 `mutex`/`condition_variable` 同步原语，再到高层 `std::async`/`future`/`promise` 异步模型和线程池实现。

## 要点

- **线程创建四种方式**：普通函数、lambda、成员函数、函数对象 `operator()`
- **常见线程错误**：`std::ref` 传递临时变量（悬空引用）、对象提前析构导致 this 悬空（用智能指针）
- **同步层次**：`mutex`（手动 lock/unlock）→ `lock_guard`（RAII 自动）→ `unique_lock`（灵活，支持 defer/try_lock/条件变量）
- **条件变量**：`cv.wait(lock, predicate)` 自动处理虚假唤醒；生产者-消费者模型中推荐先 unlock 再处理任务
- **线程池**：任务队列 + 工作线程 + 条件变量唤醒。带返回值版本用 `packaged_task` + `future`
- **异步抽象三层**：`promise`（最低，手动 set_value）→ `packaged_task`（中等，自动关联 future）→ `async`（最高，一行启动）
- **启动策略**：`std::launch::async`（新线程立即执行）、`std::launch::deferred`（get() 时执行）。**默认是两者组合，不保证异步**
- **future vs shared_future**：`future` 的 `get()` 消耗状态（仅一次）；多线程共享结果用 `shared_future`（可拷贝，多次 `get()`）
- **promise 生命周期陷阱**：promise 必须先于 `future::get()` 析构，否则抛 `broken_promise`

## 关联页面

- [[concepts/C++并发与异步|C++ 并发与异步]] — 概念综合页
- [[concepts/CSharp并发模型|C# 并发模型]] — Task/async/await 对比
