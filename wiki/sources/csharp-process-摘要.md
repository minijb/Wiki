---
title: "C# Process 进程管理 — 摘要"
type: source-summary
updated: 2026-05-10
tags: [csharp, process, async, cancellation]
source: "raw/cs/languages/csharp-process.md"
---

# C# Process 进程管理 — 摘要

来源：`raw/cs/languages/csharp-process.md`

## 概述

`System.Diagnostics.Process` 类的完整使用指南，重点在 stdout/stderr 并发读取防死锁，以及 `WaitForExitAsync` + `CancellationToken` 的协作式进程控制。

## 要点

- **两种启动方式**：`Process.Start()` 静态便捷方法 vs 构建 `ProcessStartInfo` + `Process` 对象精细控制。`UseShellExecute` 与 `RedirectStandard*` 互斥
- **死锁陷阱**：stdout 和 stderr 必须并发读取（`ReadToEndAsync` 并行执行），否则任一缓冲区满（4KB）会导致子进程阻塞、父进程卡住
- **WaitForExitAsync（.NET 5+）**：配合 `CancellationToken` 实现协作式取消。取消 Token **不会自动杀进程**，必须显式 `process.Kill()`
- **Kill 进程树（.NET 6+）**：`process.Kill(entireProcessTree: true)` 避免子进程脱离后继续运行
- **CreateLinkedTokenSource**：同时监听外部 Token 和内部超时，`when` 过滤器区分超时 vs 外部取消
- **生产级封装**：并发读取 stdout/stderr → `WaitForExitAsync` + 超时 → 超时时 `Kill` → 返回 `CommandResult(ExitCode, StdOut, StdErr)`
- **推荐库**：CliWrap（现代 API，原生 CancellationToken）、MedallionShell（内置超时/Kill）

## 关联页面

- [[concepts/CSharp进程管理|C# 进程管理]] — 概念综合页
- [[concepts/Python子进程管理|Python 子进程管理]] — subprocess 模块设计哲学对比
