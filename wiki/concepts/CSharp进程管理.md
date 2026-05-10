---
title: "C# 进程管理"
type: concept
updated: 2026-05-10
tags: [csharp, process, async, cancellation]
aliases: [CSharp进程管理, .NET Process, Process进程]
---

# C# 进程管理

`System.Diagnostics.Process` 类用于启动和管理外部进程。核心难点：输出重定向防死锁、协作式取消与超时、进程树清理。

## ProcessStartInfo 关键配置

| 属性 | 说明 |
|------|------|
| `FileName` | 可执行文件路径 |
| `Arguments` | 命令行参数 |
| `UseShellExecute` | 是否使用 OS Shell 启动。与 `RedirectStandard*` **互斥** — 重定向必须设为 `false` |
| `CreateNoWindow` | 不显示新窗口（仅 Windows） |
| `RedirectStandardOutput` / `RedirectStandardError` / `RedirectStandardInput` | 重定向标准流 |
| `WorkingDirectory` | 工作目录 |
| `Environment` | 环境变量字典 |

## 输出重定向死锁

这是使用 `Process` 最常见的陷阱。子进程的 stdout 和 stderr 各有约 4KB 缓冲区：

- 如果子进程写满了 stderr 缓冲区，等待父进程读取
- 父进程却卡在读取 stdout（等待子进程关闭 stdout）
- → 互相等待，死锁

**解决方案：并发读取。**

```csharp
process.Start();
var outputTask = process.StandardOutput.ReadToEndAsync();
var errorTask  = process.StandardError.ReadToEndAsync();
await process.WaitForExitAsync(cancellationToken);
var output = await outputTask;
var error  = await errorTask;
```

## 协作式取消与超时

现代 .NET 采用**协作式取消** —— 不是从外部杀线程，而是让代码主动检查 `CancellationToken` 并优雅退出。

`WaitForExitAsync`（.NET 5+）接受 `CancellationToken`，配合 `CancellationTokenSource` 实现超时：

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(2));
try
{
    await process.WaitForExitAsync(cts.Token);
}
catch (OperationCanceledException)
{
    process.Kill(entireProcessTree: true);  // 必须显式杀进程
    throw new TimeoutException("进程执行超时");
}
```

> [!warning] 取消 Token 不会自动杀进程
> 取消 `CancellationToken` 只停止等待，子进程仍在运行。超时后必须显式 `process.Kill()`。

### Kill 进程树（.NET 6+）

```csharp
process.Kill(entireProcessTree: true);
```

Linux 上尤其重要 —— 避免子进程脱离后成为孤儿进程继续运行。

### CreateLinkedTokenSource

同时监听外部取消和内部超时：

```csharp
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
linkedCts.CancelAfter(timeout.Value);
await process.WaitForExitAsync(linkedCts.Token);
```

任一来源触发，`linkedCts.Token` 都会被取消。配合 `when` 过滤器区分触发源：

```csharp
catch (OperationCanceledException) when (!cancellationToken.IsCancellationRequested)
{
    // 超时触发 → Kill 进程
}
// 外部 Token 触发 → when 为 false，异常继续上抛
```

## 生产级推荐库

| 库 | 特点 |
|----|------|
| **CliWrap** | 现代 API，`ExecuteAsync()` 原生支持 CancellationToken、管道、事件流 |
| **MedallionShell** | 内置超时/Kill、参数转义、跨平台 |

## 关键原则

- 使用 `using` 释放 Process 对象
- `WaitForExitAsync(CancellationToken)` 而非同步 `WaitForExit`
- stdout 和 stderr **必须并发读取**
- 库代码用 `ConfigureAwait(false)`
- 超时后显式 `Kill(entireProcessTree: true)`

## 关联页面

- [[sources/csharp-process-摘要|C# Process 来源摘要]]
- [[concepts/Python子进程管理|Python 子进程管理]] — subprocess 设计哲学与 C# Process 对比
