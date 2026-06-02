---
title: "C++ 并发与异步"
type: concept
updated: 2026-06-02
tags: [cpp, concurrency, thread, async, future]
aliases: [C++多线程, C++异步, C++并发]
---

# C++ 并发与异步

C++ 并发编程从底层线程创建到高层异步模型，核心权衡：何时用裸 `thread`/`mutex`，何时上升到 `async`/`future`。

## 线程基础

四种创建方式：普通函数、lambda、成员函数（`&Worker::run, &w, args...`）、函数对象（重载 `operator()`）。

必须 `join()` 或 `detach()`（二选一，否则 `std::terminate`）。常见坑：`std::ref` 传临时变量导致悬空引用、对象提前析构导致 this 悬空——用 `shared_ptr` 解决。

## 同步原语

```
mutex → lock_guard → unique_lock
 手动      自动RAII    灵活（defer/try_lock/条件变量）
```

`std::call_once` + `std::once_flag` 保证多线程环境下仅执行一次（单例初始化）。

条件变量配合 `unique_lock` 实现生产者-消费者模式：

```cpp
cv.wait(lock, [] { return !queue.empty(); });  // 自动处理虚假唤醒
```

## 异步模型三层抽象

| 抽象层 | 组件 | 何时使用 |
|--------|------|---------|
| 最低 | `std::promise` + `std::future` | 需要手动控制结果设置时机 |
| 中等 | `std::packaged_task` | 包装已有函数，自动关联 future |
| 最高 | `std::async` | 最简单的异步启动，一行搞定 |

**`std::future`** 的 `get()` 消耗状态（仅一次），`wait()` 不消耗（可多次）。多线程共享结果时用 `std::shared_future`（可拷贝，多次 `get()` 返回相同值）。

**`std::async` 默认启动策略是 `async | deferred`**——不保证异步执行！需要异步时显式传 `std::launch::async`。

## 线程池

核心模式：任务队列 + 工作线程 + 条件变量唤醒。析构时设置 stop 标志 → `notify_all()` → 逐个 `join()`。

带返回值的线程池：用 `std::packaged_task` 包装任务，`commit()` 返回 `std::future`。

## 性能检测

- Visual Studio Profiler（Windows CPU 采样）
- Valgrind / callgrind（Linux 缓存和调用链）
- perf（Linux 硬件计数器）
- gprof（CPU 时间分析）

## 关联页面

- [[sources/cpp-concurrency-摘要|C++ 并发与异步 来源摘要]]
- [[concepts/CSharp并发模型|C# 并发模型]] — Task/async/await 与 future/promise 对比
