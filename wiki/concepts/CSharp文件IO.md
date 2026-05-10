---
title: "C# 文件 I/O"
type: concept
updated: 2026-05-10
tags: [csharp, file-io, stream, async]
aliases: [CSharp文件IO, .NET文件操作]
---

# C# 文件 I/O

C# 文件操作的分层架构：从便捷静态方法到底层流控，核心在于正确使用异步 I/O 避免线程池浪费。

## 类分层与选择

| 层级 | 类 | 场景 |
|------|-----|------|
| 便捷层 | `File` / `FileInfo` | 小文件一次性读写、元数据查询 |
| 文本流 | `StreamReader` / `StreamWriter` | 逐行处理大文本 |
| 字节流 | `FileStream` | 完全控制、大文件、二进制 |
| 结构化二进制 | `BinaryReader` / `BinaryWriter` | 按基本类型读写固定布局格式 |
| 路径 | `Path` | 路径合并、扩展名提取 |

`Directory` / `DirectoryInfo` 负责目录的创建、枚举和遍历。

## 真正的异步 I/O

这是 C# 文件 I/O 中最常见的陷阱。

`FileStream` 构造时不传 `useAsync: true`（或 `FileOptions.Asynchronous`），`ReadAsync` 等异步方法退化为**线程池线程执行同步 I/O** —— 表面上不阻塞调用线程，实际上多占用了一个线程池线程去干同步活，白白增加调度开销。

```csharp
// 真正的异步 I/O：Windows IOCP / Linux io_uring
await using var fs = new FileStream(
    path, FileMode.Open, FileAccess.Read,
    FileShare.Read, bufferSize: 4096,
    FileOptions.Asynchronous);
await fs.ReadAsync(buffer);
```

便捷方法（`File.OpenText`、`File.CreateText`、`File.OpenRead`、`File.OpenWrite`）内部不传 `FileOptions.Asynchronous`，不提供真正的异步 I/O。需要时手动 `new FileStream(...)` 并传入 `FileOptions.Asynchronous`。

## 编码

.NET 默认 `Encoding.UTF8` 带 BOM。跨平台场景（Linux/macOS/Web API）推荐显式使用无 BOM UTF-8：

```csharp
private static readonly Encoding Utf8NoBom = new UTF8Encoding(false);
```

## 缓冲区优化

- 默认 4096 字节（与 NTFS 分配单元对齐），大文件顺序读写可用 8192
- `ArrayPool<T>.Shared.Rent()` 复用缓冲区，减少 GC 分配
- `Span<T>` 零堆分配切片，但不能跨越 `await` 边界；`Memory<T>` 可用于异步

## FileShare 与 FileOptions

- `FileShare.Read` — 允许多进程同时读取（日志查看场景）
- `FileShare.None` — 独占访问
- `FileOptions.SequentialScan` — 提示顺序读取，OS 预读到更大缓存
- `FileOptions.RandomAccess` — 提示随机访问，减少预读
- `FileOptions.DeleteOnClose` — 关闭时自动删除（临时文件）
- `FileOptions.WriteThrough` — 绕过 OS 缓存直接写磁盘

## 决策速查

```
< 10 MB   → File.ReadAllTextAsync / File.WriteAllTextAsync
10-100 MB → StreamReader.ReadLineAsync
> 100 MB  → FileStream + ReadAsync + ArrayPool
文本格式  → StreamReader / StreamWriter
二进制    → FileStream 直读 byte[] / BinaryReader
JSON     → JsonSerializer.SerializeAsync
日志追加  → File.AppendAllTextAsync
```

## 常见错误

| 错误 | 后果 | 修复 |
|------|------|------|
| `FileStream` 不传 `FileOptions.Asynchronous` | 伪异步，线程池阻塞 | 显式传参 |
| 大文件用 `ReadAllText` | `OutOfMemoryException` | `StreamReader` 逐行读 |
| `await` 后忘记 `Dispose` | 文件句柄泄漏 | `await using var` |
| 同步上下文 `.Result` / `.Wait()` | 死锁 | 全程 `async/await` |

## 关联页面

- [[sources/csharp-file-io-摘要|C# 文件 I/O 来源摘要]]
- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib/os/shutil 分层架构对比
