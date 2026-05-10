---
title: "C# Process 进程管理"
date: 2026-05-10
tags: [csharp, process, system, async]
type: language
aliases: [Process进程, 进程管理]
description: C# Process 类进程管理：启动、参数配置、输出重定向、异步等待、超时处理、Python 互操作
---

# C# Process 进程管理

## Process 概述

`Process` 类位于 `System.Diagnostics` 命名空间，用于启动和管理外部进程。它封装了进程的线程集、加载的模块（.dll 和 .exe 文件）以及性能信息（如内存使用量）。

此类型实现 `IDisposable`，使用后必须释放。推荐使用 `using` 语句。

## 两种启动方式

`Process.Start()` 静态方法内部同样构建 `ProcessStartInfo`，以下分类是**便捷方式 vs 精细控制**，并非对等的两种机制。

### 1. 使用 Process 对象（精细控制）

```csharp
try
{
    using (Process myProcess = new Process())
    {
        myProcess.StartInfo.UseShellExecute = false;
        myProcess.StartInfo.FileName = "C:\\HelloWorld.exe";
        myProcess.StartInfo.CreateNoWindow = true;
        myProcess.Start();
    }
}
catch (Exception e)
{
    Console.WriteLine(e.Message);
}
```

### 2. 静态 Process.Start（便捷启动）

```csharp
// 直接启动可执行文件
Process.Start("IExplore.exe");

// 带参数启动
Process.Start("IExplore.exe", "www.example.com");
Process.Start("IExplore.exe", "C:\\myPath\\myFile.htm");
```

## ProcessStartInfo 配置

指定进程启动时的参数集合。常用属性：

| 属性 | 说明 |
|------|------|
| `FileName` | 要启动的应用程序或文档路径 |
| `Arguments` | 命令行参数 |
| `UseShellExecute` | 是否使用操作系统 Shell 启动。与 `RedirectStandard*` **互斥**——重定向必须设为 `false` |
| `CreateNoWindow` | 是否在新窗口中启动（仅 Windows 有效） |
| `WorkingDirectory` | 工作目录 |
| `RedirectStandardOutput` | 重定向标准输出流 |
| `RedirectStandardError` | 重定向标准错误流 |
| `RedirectStandardInput` | 重定向标准输入流 |
| `StandardOutputEncoding` | 标准输出编码 |
| `StandardErrorEncoding` | 标准错误编码 |
| `Environment` | 进程环境变量 |

### 通过 ProcessStartInfo 启动

```csharp
ProcessStartInfo startInfo = new ProcessStartInfo("IExplore.exe")
{
    WindowStyle = ProcessWindowStyle.Minimized,
    Arguments = "www.example.com"
};
Process.Start(startInfo);
```

## 异步等待与超时（现代做法）

### CancellationToken：协作式取消

`CancellationToken` 是 .NET 中取消操作的标准机制 —— 一个轻量级结构体，在线程/任务之间传递"该停止了"的信号。

**核心模型：**

```
调用方（持有 CancellationTokenSource）          被调用方（接收 CancellationToken）
     │                                                     │
     │  cts.Cancel() / cts.CancelAfter(timeout)            │
     │ ─────────────────────────────────────────────────►  │
     │                                                     │
     │                                      WaitForExitAsync(token)
     │                                      token 被取消 → 抛出 OperationCanceledException
```

**关键特性：**

- **它只传递信号** —— `Cancel()` 不会自动杀进程、不会中断代码。纯粹把 `IsCancellationRequested` 设为 `true`
- **响应靠被调用方** —— `WaitForExitAsync(token)` 内部注册了 Token 的回调；Token 一取消就抛 `OperationCanceledException`
- **进程不会自己停** —— Token 取消只停止了等待，子进程仍在运行。超时后 **必须显式 `process.Kill()`**
- **协作式 = 安全** —— 不像 `Thread.Abort` 那样暴力中断，保证了 `finally` 块和 `using` 清理总能执行

### WaitForExitAsync（.NET 5+）

```csharp
using var process = new Process
{
    StartInfo = new ProcessStartInfo
    {
        FileName = "some-tool.exe",
        Arguments = "--verbose",
        UseShellExecute = false
    }
};

process.Start();
await process.WaitForExitAsync(cancellationToken);
```

> [!note] ExitCode 时机
> 如果进程已经退出（`HasExited == true`），`WaitForExitAsync` 会立即返回。`ExitCode` 仅在进程退出后可用——提前读取会抛 `InvalidOperationException`。

### 带超时的模式

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(2));

try
{
    await process.WaitForExitAsync(cts.Token);

    if (process.ExitCode != 0)
    {
        // 处理非零退出码
    }
}
catch (OperationCanceledException)
{
    process.Kill(entireProcessTree: true);
    throw new TimeoutException("进程执行超时");
}
```

> [!warning] 取消 Token 不会自动杀进程
> 取消 `CancellationToken` 只停止等待，**不会杀死进程**；必须显式调用 `process.Kill()`。

### Kill 整个进程树（.NET 6+）

```csharp
process.Kill(entireProcessTree: true);
```

在 Linux 上尤其重要，可避免子进程脱离后继续运行。Windows 上依赖进程树追踪（Job Object），效果因进程类型而异——控制台进程通常有效，GUI 进程可能不受影响。

## 输出重定向：并发读取防死锁

这是使用 Process 最常见的陷阱。

```csharp
// ❌ 错误：顺序读取可能导致死锁
var output = await process.StandardOutput.ReadToEndAsync();
var error  = await process.StandardError.ReadToEndAsync();  // 可能永远等不到
await process.WaitForExitAsync();

// ✅ 正确：并发读取 stdout 和 stderr
process.Start();

var outputTask = process.StandardOutput.ReadToEndAsync();
var errorTask  = process.StandardError.ReadToEndAsync();

await process.WaitForExitAsync(cancellationToken);

var output = await outputTask;
var error  = await errorTask;
```

**死锁原理：** 子进程可能填满 stderr 缓冲区（4KB）后阻塞，而父进程卡在读取 stdout —— 互相等待，永远无法完成。

## 生产级封装

```csharp
public static async Task<CommandResult> RunCommandAsync(
    string fileName,
    string arguments,
    CancellationToken cancellationToken = default,
    TimeSpan? timeout = null)
{
    using var process = new Process
    {
        StartInfo = new ProcessStartInfo
        {
            FileName = fileName,
            Arguments = arguments,
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true
        },
        EnableRaisingEvents = true  // 允许 Exited 事件触发；配合 WaitForExitAsync 时非必需但无害
    };

    process.Start();

    var outputTask = process.StandardOutput.ReadToEndAsync();
    var errorTask  = process.StandardError.ReadToEndAsync();

    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    if (timeout.HasValue)
        linkedCts.CancelAfter(timeout.Value);

    try
    {
        await process.WaitForExitAsync(linkedCts.Token).ConfigureAwait(false);
    }
    catch (OperationCanceledException) when (!cancellationToken.IsCancellationRequested)
    {
        process.Kill(entireProcessTree: true);
        await process.WaitForExitAsync();
    }

    return new CommandResult(
        process.ExitCode,
        await outputTask.ConfigureAwait(false),
        await errorTask.ConfigureAwait(false)
    );
}

public record CommandResult(int ExitCode, string StdOut, string StdErr);
```

### `CreateLinkedTokenSource` 的作用

这个方法封装同时监听**两个取消来源**：

1. **外部 `cancellationToken`** — 调用方可能随时取消
2. **内部超时** — `timeout` 到期后自动取消

但 `CancellationToken` 是只读的，无法直接调用 `CancelAfter`。所以 `CreateLinkedTokenSource` 创建一个新的 `CancellationTokenSource`，将它与外部 Token 链接：

```csharp
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
linkedCts.CancelAfter(timeout.Value);  // 超时时也取消 linkedCts.Token
```

**任一来源触发，`linkedCts.Token` 都会被取消** —— 用一个 Token 统一监听两个来源。

catch 块中的 `when` 过滤器配合区分"是谁触发"：

```csharp
catch (OperationCanceledException) when (!cancellationToken.IsCancellationRequested)
```

- **`when` 为 true** → 外部 Token 未被取消，说明是**超时**触发 → kill 进程
- **`when` 为 false** → 外部 Token 已被取消 → 异常不匹配，继续往上抛，由调用方处理

## 应用：C# 运行 Python 脚本

```csharp
public static async Task RunPythonAsync(string filePath, string args)
{
    var startInfo = new ProcessStartInfo
    {
        FileName = "python",
        Arguments = $"{filePath} {args}",
        UseShellExecute = false,
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        WorkingDirectory = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "../../")
        // 注意：Unity 环境下请使用 Application.dataPath 替代
    };

    using var process = new Process { StartInfo = startInfo };
    process.Start();

    var outputTask = process.StandardOutput.ReadToEndAsync();
    var errorTask  = process.StandardError.ReadToEndAsync();

    await process.WaitForExitAsync();

    var output = await outputTask;
    var error  = await errorTask;

    if (process.ExitCode != 0)
    {
        throw new InvalidOperationException($"Python 执行失败: {error}");
    }
}
```

## 推荐库

对于生产环境，考虑使用封装完善的第三方库：

| 库 | 特点 |
|----|------|
| **CliWrap** | 现代 API，`ExecuteAsync()` 原生支持 CancellationToken、管道、事件流 |
| **MedallionShell** | 内置超时/Kill、参数转义、跨平台 |

这些库抽象了底层的死锁陷阱和平台差异。

## 关键原则

- 始终使用 `using` 释放 Process 对象
- 使用 `WaitForExitAsync(CancellationToken)` 而非同步 `WaitForExit`
- stdout 和 stderr **必须并发读取**
- 库代码中使用 `ConfigureAwait(false)`
- 超时后必须显式 `Kill(entireProcessTree: true)`

## 参见

- [[concepts/Python子进程管理|Python 子进程管理]] — subprocess 模块的设计哲学与 C# Process 的对比
