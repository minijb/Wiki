---
title: "C# 文件 I/O 处理 — 摘要"
type: source-summary
updated: 2026-05-10
tags: [csharp, file-io, stream, async]
source: "raw/cs/languages/csharp-file-io.md"
---

# C# 文件 I/O 处理 — 摘要

来源：`raw/cs/languages/csharp-file-io.md`

## 概述

C# 文件 I/O 的完整分层指南，从 `File.ReadAllText` 便捷 API 到底层 `FileStream` 流控，核心警示"真正的异步 I/O"这一最易错点。

## 要点

- **类分层**：`File`/`FileInfo`（一次性/元数据）、`Directory`/`DirectoryInfo`、`Path`（路径字符串）、`FileStream`（底层字节流）、`StreamReader`/`StreamWriter`（文本流）、`BinaryReader`/`BinaryWriter`（结构化二进制）
- **真正的异步 I/O**：创建 `FileStream` 时必须传 `FileOptions.Asynchronous`（或 `useAsync: true`），否则 `ReadAsync` 退化为线程池模拟同步 I/O，白白增加调度开销
- **stdout/stderr 并发读取**：顺序读取可能导致缓冲区满的死锁
- **编码**：.NET 默认 UTF-8 with BOM，跨平台推荐 `new UTF8Encoding(false)` 显式无 BOM
- **缓冲区优化**：`ArrayPool<T>.Shared.Rent()` 复用缓冲区；`Span<T>` 零堆分配切片（不能跨越 await），`Memory<T>` 可安全用于异步
- **决策树**：<10MB 用 `ReadAllTextAsync`，10-100MB 用 `StreamReader.ReadLineAsync`，>100MB 用 `FileStream` + `ReadAsync` + `ArrayPool`

## 关联页面

- [[concepts/CSharp文件IO|C# 文件 I/O]] — 概念综合页
- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib/os/shutil 分层架构对比
