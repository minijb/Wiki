---
title: "C++ 并发与异步编程"
date: 2026-06-02
tags: [cpp, concurrency, thread, async, future, mutex]
type: language
aliases: [C++多线程, C++线程, C++并发, C++异步]
description: C++ 并发编程完整指南：std::thread 创建与管理、mutex/lock_guard/unique_lock 同步原语、condition_variable 条件变量、线程池实现、std::async/future/promise/packaged_task 异步模型
---

# C++ 并发与异步编程

## 线程创建与管理

### 四种启动方式

```cpp
#include <thread>

// 1. 普通函数
void threadFunc(int n) { /* ... */ }
std::thread t1(threadFunc, 42);

// 2. Lambda
std::thread t2([](int n) { /* ... */ }, 42);

// 3. 成员函数
class Worker {
public:
    void run(int n) { /* ... */ }
};
Worker w;
std::thread t3(&Worker::run, &w, 42);

// 4. 函数对象（重载 operator()）
struct Task {
    void operator()(int n) { /* ... */ }
};
std::thread t4(Task{}, 42);
```

### join 与 detach

```cpp
std::thread t(func);

// join：等待线程结束（阻塞当前线程）
if (t.joinable()) {
    t.join();
}

// detach：分离线程，使其独立运行（主线程不等待）
// ⚠️ 危险：主线程结束后分离线程可能被强制终止
t.detach();
```

### 常见错误

| 错误场景 | 后果 | 预防 |
|----------|------|------|
| `std::ref(x)` 传递临时变量 | 悬空引用，未定义行为 | 确保引用对象生命周期覆盖线程 |
| 线程访问已释放的局部变量 | 悬空指针/引用 | 使用智能指针或确保变量在线程结束前存活 |
| 传递裸指针后释放 | 线程访问野指针 | 使用 `shared_ptr` 共享所有权 |
| 类成员函数作为入口，类对象提前析构 | this 指针悬空 | 使用 `shared_from_this()` 或确保对象生命周期 |
| 入口函数为 `private` 成员 | 编译错误 | 封装启动方法或使用友元 |

## 同步原语

### std::mutex

最基础的互斥量，手动加锁/解锁。

```cpp
std::mutex mtx;
int counter = 0;

void increment() {
    mtx.lock();
    counter++;
    mtx.unlock();  // ⚠️ 必须解锁，异常时可能未执行
}
```

### std::lock_guard

RAII 封装，构造时自动锁定，析构时自动解锁。不可拷贝，不可移动，仅限局部作用域。

```cpp
std::mutex mtx;

void safeFunc() {
    std::lock_guard<std::mutex> guard(mtx);
    // 临界区 ... 发生异常也会自动解锁
}
```

### std::unique_lock

更灵活的 RAII 封装，支持延迟加锁、条件变量、超时操作。

```cpp
std::timed_mutex mtx;
std::unique_lock<std::timed_mutex> lock(mtx, std::defer_lock);  // 构造时不锁定

// 方法
lock.lock();           // 阻塞直到获取锁
lock.try_lock();       // 非阻塞，返回 true/false
lock.try_lock_for(     // 阻塞指定时长
    std::chrono::seconds(1));
lock.try_lock_until(   // 阻塞到指定时间点
    std::chrono::steady_clock::now() + std::chrono::seconds(1));
lock.unlock();         // 提前解锁
```

### std::timed_mutex

支持 `try_lock_for` / `try_lock_until` 的互斥量变体，配合 `std::unique_lock` 使用。

### std::recursive_mutex

同一线程可以多次 lock，需要匹配次数的 unlock。

## std::call_once

确保函数在多线程环境中只被执行一次，常用于单例的线程安全初始化。

```cpp
class Logger {
    static Logger* instance;
    static std::once_flag initFlag;

    static void init() {
        instance = new Logger();
    }

public:
    static Logger& getInstance() {
        std::call_once(initFlag, init);
        return *instance;
    }
};
```

> `call_once` 仅适用于多线程场景，单线程中直接调用函数即可。

## 条件变量 (condition_variable)

### 基本用法

```cpp
std::queue<int> g_queue;
std::condition_variable g_cv;
std::mutex g_mtx;

void Producer() {
    for (int i = 0; i < 10000; i++) {
        {
            std::unique_lock<std::mutex> lock(g_mtx);
            g_queue.push(i);
        }
        g_cv.notify_one();  // 通知一个等待线程
    }
}

void Consumer() {
    while (true) {
        std::unique_lock<std::mutex> lock(g_mtx);

        // wait 接受两个参数：锁 + 条件谓词
        // 等价于：while (!pred()) { wait(lock); }
        g_cv.wait(lock, [] { return !g_queue.empty(); });

        int value = g_queue.front();
        g_queue.pop();
        lock.unlock();  // 提前解锁，避免处理任务时持锁

        std::cout << "Processing: " << value << std::endl;
    }
}
```

### 使用步骤

1. 创建 `std::condition_variable` 对象
2. 创建 `std::mutex` 对象
3. 在等待线程中：获取 `std::unique_lock`，调用 `wait()` / `wait_for()` / `wait_until()`
4. 在通知线程中：调用 `notify_one()` 或 `notify_all()`

### wait 的两种形式

```cpp
// 形式 1：无条件等待
cv.wait(lock);

// 形式 2：带谓词（推荐）
// 自动处理虚假唤醒：while (!pred()) { wait(lock); }
cv.wait(lock, [] { return ready; });
```

## 线程池

### 简单实现

```cpp
#include <condition_variable>
#include <functional>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

class SimpleThreadPool {
public:
    SimpleThreadPool(size_t numThreads) : stop(false) {
        for (size_t i = 0; i < numThreads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx);
                        cv.wait(lock, [this] {
                            return stop || !tasks.empty();
                        });

                        if (stop && tasks.empty())
                            return;

                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    ~SimpleThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            stop = true;
        }
        cv.notify_all();
        for (auto& worker : workers) {
            worker.join();
        }
    }

    template <class F, class... Args>
    void addTask(F&& f, Args&&... args) {
        auto task = std::bind(std::forward<F>(f), std::forward<Args>(args)...);
        {
            std::lock_guard<std::mutex> lock(mtx);
            tasks.emplace(std::move(task));
        }
        cv.notify_one();
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop;
};
```

### 带返回值的线程池

使用 `std::packaged_task` 包装任务，通过 `std::future` 返回结果：

```cpp
template <class F, class... Args>
auto commit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
    using RetType = decltype(f(args...));

    auto task = std::make_shared<std::packaged_task<RetType()>>(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...));

    std::future<RetType> ret = task->get_future();
    {
        std::lock_guard<std::mutex> lock(mtx);
        tasks.emplace([task] { (*task)(); });
    }
    cv.notify_one();
    return ret;
}
```

## 异步编程模型

异步的核心思想：**不需要关注线程什么时候结束，只需要知道结果是否完成**。

### 抽象层次

从低到高三个抽象层次：

| 层次 | 组件 | 说明 |
|------|------|------|
| 最低 | `std::promise` | 手动设置值或异常，通过关联的 `future` 获取 |
| 中等 | `std::packaged_task` | 包装可调用对象，返回值自动存入 `future` |
| 最高 | `std::async` | 一行启动异步任务，自动管理线程/promise/future |

### std::async

```cpp
#include <future>
#include <chrono>

std::string fetchDataFromDB(std::string query) {
    std::this_thread::sleep_for(std::chrono::seconds(5));
    return "Data: " + query;
}

int main() {
    // 异步执行，返回 future
    std::future<std::string> result =
        std::async(std::launch::async, fetchDataFromDB, "Data");

    // 主线程继续其他工作
    std::cout << "Doing something else..." << std::endl;

    // 需要结果时阻塞获取
    std::string dbData = result.get();
    std::cout << dbData << std::endl;
}
```

`std::async` 内部自动创建 `std::promise`，并将任务函数的返回值存入 `std::future`。

### 启动策略

| 策略 | 行为 |
|------|------|
| `std::launch::async` | 立即在新线程上异步执行 |
| `std::launch::deferred` | 延迟执行：在调用 `get()` 或 `wait()` 时才执行（同步） |
| `std::launch::async \| std::launch::deferred`（默认） | 由实现决定，可能异步也可能延迟 |

> **注意**：默认策略的行为取决于编译器实现。如果依赖异步执行，显式指定 `std::launch::async`。

### std::future

`std::future` 代表一个尚未就绪的异步操作结果。

```cpp
std::future<int> fut = std::async(std::launch::async, [] { return 42; });

// 阻塞获取结果（仅能调用一次！）
int value = fut.get();

// 仅等待完成，不获取结果（可多次调用）
fut.wait();

// 非阻塞检查状态
if (fut.wait_for(std::chrono::seconds(0)) == std::future_status::ready) {
    // 结果已就绪
}

// 等待到指定时间点
if (fut.wait_until(deadline) == std::future_status::ready) {
    // 结果已就绪
}
```

**`get()` vs `wait()`**：
- `get()`：阻塞等待 + 返回结果 + **消耗** future 状态（只能调用一次）
- `wait()`：纯阻塞等待，不消耗状态（可多次调用）

### std::promise

手动在某线程设置值，另一个线程通过 `future` 获取。

```cpp
void setValue(std::promise<int> prom) {
    prom.set_value(10);  // 设置结果
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();

    std::thread t(setValue, std::move(prom));

    std::cout << "Waiting..." << std::endl;
    std::cout << "Result: " << fut.get() << std::endl;  // 阻塞直到 set_value

    t.join();
}
```

**异常传递**：通过 `set_exception()` 传递异常：

```cpp
void mayThrow(std::promise<void> prom) {
    try {
        throw std::runtime_error("An error occurred!");
    } catch (...) {
        prom.set_exception(std::current_exception());
    }
}

// 消费端
try {
    fut.get();
} catch (const std::exception& e) {
    std::cout << "Exception: " << e.what() << std::endl;
}
```

**promise 生命周期陷阱**：如果 `promise` 先于 `future::get()` 被销毁，`get()` 会抛出 `std::future_error`（`broken_promise`）。

```cpp
// ❌ 危险：prom 在 get() 前可能被销毁
std::future<int> fut;
{
    std::promise<int> prom;
    fut = prom.get_future();
    std::thread t(setValue, std::move(prom));
}  // prom 离开作用域
fut.get();  // 可能抛出 future_error

// ✅ 正确：确保 promise 生命周期覆盖 get()
```

### std::packaged_task

包装可调用对象，将其返回值自动关联到 `future`。

```cpp
int myTask() {
    std::this_thread::sleep_for(std::chrono::seconds(5));
    return 42;
}

void usePackagedTask() {
    std::packaged_task<int()> task(myTask);
    std::future<int> result = task.get_future();

    // 移动到线程执行
    std::thread t(std::move(task));
    t.detach();

    int value = result.get();  // 阻塞获取
    std::cout << "Result: " << value << std::endl;
}
```

使用步骤：
1. 创建 `std::packaged_task` 包装任务
2. 调用 `get_future()` 获取关联的 `std::future`
3. 在另一个线程上调用 `operator()` 执行
4. 需要时调用 `future::get()` 获取结果

### std::shared_future

当**多个线程**需要等待同一个异步操作结果时使用。`std::future` 只能移动（`get()` 消耗状态），`std::shared_future` 可以拷贝，允许多次 `get()`。

```cpp
std::promise<int> promise;
std::shared_future<int> sf = promise.get_future();

std::thread t1([sf] { std::cout << sf.get() << std::endl; });
std::thread t2([sf] { std::cout << sf.get() << std::endl; });
std::thread t3([sf] { std::cout << sf.get() << std::endl; });

std::thread producer([&promise] {
    promise.set_value(42);
});

producer.join();
t1.join();
t2.join();
t3.join();
```

> `shared_future` 可以拷贝，多次 `get()` 都返回相同结果。

**常见错误**：同一个 `future` 被 `std::move` 两次后传给线程。第一次移动后源对象失效，第二次导致 `no state` 错误。

## 性能检测工具

| 工具 | 平台 | 用途 |
|------|------|------|
| Visual Studio Profiler | Windows | CPU 采样、内存分配、线程分析 |
| Valgrind (callgrind/cachegrind) | Linux | 缓存命中、函数调用链分析 |
| gprof | Linux | CPU 时间分析 |
| perf | Linux | 硬件性能计数器 |
| VTune | 跨平台 | Intel CPU 深度分析 |

## 异步组件速查

| 组件 | 作用 | 创建 | 获取结果 |
|------|------|------|---------|
| `std::thread` | 创建线程 | 构造函数 | `join()` 等待结束 |
| `std::async` | 异步任务 | `std::async(policy, func, args...)` | `future::get()` |
| `std::packaged_task` | 包装任务+future | `std::packaged_task<Ret(Args)>` | `task.get_future().get()` |
| `std::promise` | 手动设置结果 | 构造函数 | `prom.get_future().get()` |
| `std::future` | 获取异步结果 | 多种来源 | `get()`（仅一次） |
| `std::shared_future` | 多线程共享结果 | `future.share()` | `get()`（多次） |

## 参见

- [[concepts/CSharp并发模型|C# 并发模型]] — Task、async/await 与 C++ future/promise 对比
