---
title: "C# vs C++ 异步模型"
type: comparison
updated: 2026-06-02
tags: [csharp, cpp, async, concurrency, comparison]
---

# C# vs C++ 异步模型

对比 C# 的 TAP（Task-based Asynchronous Pattern）与 C++ 的 `std::future`/`std::async`/C++20 协程，从语法表现力、运行时开销、取消机制、错误处理到生态成熟度。

## 语法对比

### C# TAP: async/await

```csharp
public async Task<string> FetchAsync(string url, CancellationToken ct = default)
{
    using var client = new HttpClient();
    var response = await client.GetAsync(url, ct);
    return await response.Content.ReadAsStringAsync(ct);
}
```

编译器将 `async` 方法展开为 `IAsyncStateMachine` 结构体，每个 `await` 是一个状态切换点。状态机在首次未完成 `await` 时装箱到堆，后续 `await` 在堆上执行。

### C++: std::future/std::async

```cpp
std::future<std::string> FetchAsync(const std::string& url) {
    return std::async(std::launch::async, [&] {
        // 使用 libcurl 或其他 HTTP 库（C++ 标准库无 HTTP 客户端）
        return Fetch(url);
    });
}

auto result = future.get();  // 阻塞等待
```

C++20 协程：

```cpp
Task<std::string> FetchAsync(const std::string& url) {
    auto response = co_await HttpGetAsync(url);
    co_return response.Body();
}
```

| 维度 | C# | C++ |
|------|-----|-----|
| 语法糖 | `async`/`await`（C# 5.0，2012） | `co_await`/`co_return`（C++20，2020） |
| 返回类型 | `Task<T>` / `ValueTask<T>` / `IAsyncEnumerable<T>` | 自定义 promise_type + future 类型；无标准库 task 抽象 |
| 编译器状态机 | 成熟的 `IAsyncStateMachine`，struct 为默认 | 无栈协程，由 `promise_type` 控制分配 |
| I/O 完成 | IOCP + `TaskCompletionSource` 无阻塞等待 | 需第三方库（ASIO `async_read`）绑定协程 |

## 性能模型

### C#: 堆分配与 ValueTask 优化

- **`Task<T>`**：未完成时堆分配（Task + Delegate + 状态机装箱），完成后可缓存（`Task.CompletedTask`、`Task.FromResult`）
- **`ValueTask<T>`**：struct 包装，同步完成时零分配；异步时包装成 Task
- **`IAsyncEnumerable<T>`**：流式异步迭代，每次 `yield return` 可 `await`
- **SynchronizationContext**：控制 `await` 后回到哪个线程，UI 线程调度零成本

### C++: 自定义分配器与零开销抽象

- `std::async` 内部分配 `std::promise` + `std::future` 共享状态（堆分配）
- C++20 协程帧**可自定义分配器**，支持栈/arena 分配，理论上能做到真正的零堆分配
- 无内置调度器概念——协程 `co_await` 恢复线程由 awaiter 决定
- ASIO 的 `co_await` 适配 I/O 完成端口，性能与 C# IOCP 同级

> [!note] 通用型 vs 可配置型
> C# 的路是"通用优先"——`Task` 适用于 90% 场景，`ValueTask` 覆盖高频路径。C++ 的路是"可配置优先"——每一步分配都可自定义，但标准库提供的开箱即用能力偏少。

## 取消机制

### C#: CancellationToken

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await FetchAsync(url, cts.Token);  // 超时自动取消
```

- `CancellationToken` **贯穿整个调用链**，每个异步方法都传递它
- 取消是**协作式**的：被调用方通过 `ct.ThrowIfCancellationRequested()` 或注册 `ct.Register()` 回调来响应
- `CancellationTokenSource.CancelAfter()` / `CreateLinkedTokenSource()` 提供组合取消
- 生态高度统一——所有 BCL 异步 API 都接受 `CancellationToken`

### C++: 碎片化

- **C++20 `std::stop_token`**：最接近 `CancellationToken` 的标准方案，配合 `std::jthread` 使用
- **原子变量 + 轮询**：`std::atomic<bool> stop{false}`——传统 C++ 惯用法，无传播机制
- **`future::get()` 不可取消**：`.get()` 调用方只能等，无超时/取消支持（`future::wait_for` 只是轮询）
- **ASIO 的 `cancellation_slot`**：与 ASIO 深度绑定，非标准

| 特性 | C# | C++ |
|------|-----|-----|
| 标准化取消令牌 | `CancellationToken`（.NET 4.0，2010） | `std::stop_token`（C++20，2020） |
| 传播方式 | 方法签名传递，编译器/分析器强制检查 | 无编译期强制 |
| 超时取消 | `CancelAfter()` 内置 | 需自行组合 timer |
| 生态一致性 | 高（BCL/ASP.NET/EF Core 全遵循） | 低（各库自有方案） |

## 错误处理

### C#: try-catch 自然适配

```csharp
try
{
    var data = await FetchAsync(url, ct);
}
catch (HttpRequestException ex)
{
    // 直接捕获——await 会解包 AggregateException
}
catch (OperationCanceledException)
{
    // 取消异常
}
```

- `await` 自动解包 `AggregateException`，抛出第一个内部异常——与同步 `try-catch` 一致的体验
- `Task.Exception` 保存完整异常树（调试时可用）
- `Task.IsFaulted` / `Task.IsCanceled` 区分失败与取消

### C++: future.get() 重抛异常

```cpp
try {
    auto result = future.get();  // 阻塞；异常在此重抛
} catch (const std::exception& e) {
    // future 内部存储了 std::exception_ptr
}
```

- `std::future::get()` 在调用处重抛异常——需在消费端而非创建端捕获
- C++20 协程 `co_await` 异常传播由 `promise_type::unhandled_exception()` 控制，行为取决于库实现
- 无内置区分"取消"与"失败"——自定义异常类或用 `std::expected<T,E>`

## 生态与标准库支持

### C#: 全栈 TAP

.NET 的每一层都绑定 TAP：

- **BCL**：`Stream.ReadAsync`、`Socket.ReceiveAsync`、`DbConnection.OpenAsync`——所有 I/O API 均有 `Async` 后缀版本
- **ASP.NET Core**：请求管道全异步，同步调用会阻塞线程池导致饥饿
- **EF Core**：所有数据库操作均为 TAP
- **System.Threading.Channels**：异步生产者-消费者管道
- **PLINQ / Parallel.ForEachAsync**：数据并行 + 异步融合

### C++: 多套方案并存

- **`std::async`**：最简单但不可靠（默认不保证异步执行）
- **`std::thread` + `std::future`**：手动管理生命周期
- **ASIO + C++20 协程**：高性能 I/O，接近 C# TAP 体验但非标准
- **Boost.Fiber / Boost.Coroutine2**：栈式协程，不同范式
- **TBB / taskflow**：任务并行框架

> [!warning] C++ 异步生态的碎片化
> 没有类似 .NET BCL 的统一异步层。每个 I/O 库（ASIO、libuv、libcurl）有自己的异步模型，互不兼容。实际项目通常锁定一套生态（如 ASIO + 协程），不同生态间整合成本高。

## 选型指南

| 场景 | 推荐 |
|------|------|
| 应用层业务代码、Web 后端 | C# TAP——开箱即用，生态完整 |
| 游戏引擎底层、高频交易、嵌入式 | C++ 协程 + ASIO——可控分配，零额外 GC |
| 跨平台 I/O 密集型服务 | 两者皆可；C# 开发效率高，C++ 性能上限高 |
| 已有大量同步 C++ 代码的渐进式改造 | C++20 协程——无需重写架构 |
| 团队无 C++ 模板元编程经验 | C#——学习曲线平缓 |

## 参见

- [[CSharp异步模型]] — C# TAP 深度解析：状态机、Awaiter 模式、ConfigureAwait、SynchronizationContext
- [[C++并发与异步]] — C++ 并发三层抽象：thread/mutex → future/async → C++20 协程
- [[CSharp并发模型]] — C# 并发演进：Thread → Task → async/await → Channel
- [[CSharp内存GC]] — ValueTask 减少堆分配的原理与使用场景
