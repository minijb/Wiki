---
title: "C# 文件 I/O 处理"
date: 2026-05-10
tags: [csharp, file, io, stream, async]
type: language
aliases: [文件处理, 文件读写, FileStream]
description: C# 文件 I/O 完整指南：File/FileInfo/Directory 类、同步/异步流、编码、缓冲区优化、常见错误避免
---

# C# 文件 I/O 处理

## 核心类概览

| 类 | 用途 | 推荐场景 |
|----|------|---------|
| `File` | 静态方法，一次性操作 | 小文件（<10MB）读写 |
| `FileInfo` | 实例方法，文件元数据操作 | 需要多次访问文件属性 |
| `Directory` | 静态方法，目录操作 | 简单目录创建/枚举 |
| `DirectoryInfo` | 实例方法，目录操作 | 需要多次访问目录属性 |
| `Path` | 路径字符串操作 | 路径合并、获取扩展名等 |
| `FileStream` | 底层字节流 | 完全控制、大文件、二进制 |
| `StreamReader` / `StreamWriter` | 文本流 | 逐行处理大文本文件 |
| `BinaryReader` / `BinaryWriter` | 结构化二进制读写 | 按基本类型（int、float、string）读写二进制数据 |

## 文件和目录基础操作

### File / FileInfo

```csharp
// File — 静态便捷方法
string content = File.ReadAllText("file.txt");
File.WriteAllText("file.txt", "Hello World");
bool exists = File.Exists("file.txt");
File.Copy("source.txt", "dest.txt");
File.Delete("temp.txt");

// FileInfo — 实例方法，适合多次操作
var fileInfo = new FileInfo("file.txt");
long size = fileInfo.Length;
DateTime modified = fileInfo.LastWriteTime;
fileInfo.CopyTo("dest.txt");
```

### Path 常用操作

```csharp
string fullPath = Path.Combine("folder", "subfolder", "file.txt");
string ext = Path.GetExtension("file.txt");          // ".txt"
string name = Path.GetFileNameWithoutExtension("file.txt"); // "file"
string dir = Path.GetDirectoryName(@"C:\a\b.txt");   // "C:\a"
string temp = Path.GetTempFileName();  // ⚠️ 立即创建文件，65535 限制，推荐用 GetRandomFileName()
string safeTemp = Path.Combine(Path.GetTempPath(), Path.GetRandomFileName());
```

### Directory / DirectoryInfo

```csharp
// Directory
Directory.CreateDirectory("newFolder");
string[] files = Directory.GetFiles("folder", "*.txt");
bool dirExists = Directory.Exists("folder");

// DirectoryInfo
var dirInfo = new DirectoryInfo("folder");
FileInfo[] fileInfos = dirInfo.GetFiles("*.txt");
```

## 流式读写

### StreamReader / StreamWriter（文本）

```csharp
// 写入
using var writer = new StreamWriter("output.txt");
await writer.WriteLineAsync("Line 1");
await writer.WriteLineAsync("Line 2");
// Dispose 时自动 Flush

// 读取（逐行，内存友好）
using var reader = new StreamReader("large.txt");
string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    Console.WriteLine(line);
}
```

### FileStream（二进制/完全控制）

```csharp
await using var fs = new FileStream(
    "data.bin",
    FileMode.Open,
    FileAccess.Read);
byte[] buffer = new byte[4096];
int bytesRead;
while ((bytesRead = await fs.ReadAsync(buffer, 0, buffer.Length)) > 0)
{
    ProcessChunk(buffer.AsSpan(0, bytesRead));
}
```

### BinaryReader / BinaryWriter（结构化二进制）

当文件格式按固定布局存储基本类型（如 RIFF/WAV 头、游戏存档、自定义封包格式）时，无需手动解析 `byte[]`：

```csharp
using var reader = new BinaryReader(File.OpenRead("data.bin"));
int version = reader.ReadInt32();
float value = reader.ReadSingle();
string name = reader.ReadString();
```

配合 `FileStream` 的 `Position` 和 `Seek` 可跳转到任意偏移读取字段。

### FileShare：多进程并发访问

当多个进程同时访问同一文件时，所有进程必须使用兼容的 `FileShare` 标志：

| 值 | 含义 |
|----|------|
| `FileShare.Read` | 允许其他进程同时读取（常见于日志查看） |
| `FileShare.Write` | 允许其他进程写入 |
| `FileShare.ReadWrite` | 允许并发读写 |
| `FileShare.None` | 独占访问，其他进程无法打开 |

```csharp
// 只读打开，同时允许其他进程也读取
using var fs = new FileStream("shared.log", FileMode.Open,
    FileAccess.Read, FileShare.Read);
```

### FileOptions：缓存与行为提示

创建 `FileStream` 时可传递 `FileOptions` 标志组合，用于提示操作系统优化方式：

| 标志 | 效果 |
|------|------|
| `FileOptions.Asynchronous` | 真正的异步 I/O（等效 `useAsync: true`） |
| `FileOptions.SequentialScan` | 提示将顺序读取，预读到更大缓存 |
| `FileOptions.RandomAccess` | 提示随机访问，减少预读 |
| `FileOptions.DeleteOnClose` | 关闭时自动删除文件（临时文件） |
| `FileOptions.Encrypted` | NTFS 文件系统级加密 |
| `FileOptions.WriteThrough` | 绕过 OS 缓存直接写入磁盘 |

## 真正的异步 I/O：关键细节

创建 `FileStream` 时**不传 `useAsync: true`** 是最常见的错误 —— 异步方法退化为"线程池线程模拟同步 I/O"，白白增加调度开销。

```csharp
// ❌ 错误：底层仍是同步 I/O
using var fs = new FileStream(path, FileMode.Open, FileAccess.Read);
await fs.ReadAsync(buffer);  // 伪异步！

// ✅ 正确：真正的操作系统级异步 I/O（Windows IOCP / Linux io_uring）
await using var fs = new FileStream(
    path,
    FileMode.Open,
    FileAccess.Read,
    FileShare.Read,
    bufferSize: 4096,
    FileOptions.Asynchronous);  // 或用 useAsync: true
await fs.ReadAsync(buffer);
```

> [!tip] 真正的异步需要手动包装
> 将 `FileStream` 手动传给 `StreamReader`/`StreamWriter` 才能获得真正的异步 I/O。

```csharp
// ✅ 真正的异步逐行读取
await using var fs = new FileStream("large.txt", FileMode.Open,
    FileAccess.Read, FileShare.Read, 4096, FileOptions.Asynchronous);
using var reader = new StreamReader(fs, Encoding.UTF8);

string? line;
while ((line = await reader.ReadLineAsync()) != null)
{
    ProcessLine(line);
}
```

## 编码处理

```csharp
// .NET 默认 Encoding.UTF8 带 BOM
// 跨平台推荐显式使用无 BOM UTF-8
private static readonly Encoding Utf8NoBom = new UTF8Encoding(false);

await using var writer = new StreamWriter("config.json", false, Utf8NoBom);
```

| 平台 | 行为 |
|------|------|
| Windows | 记事本依赖 BOM 识别 UTF-8 |
| Linux/macOS | 期望纯 UTF-8（无 BOM），BOM 可能导致脚本错误 |
| Web/API | 一律推荐无 BOM |

## 缓冲区与内存优化

### 缓冲区大小选择

| 大小 | 适用 |
|------|------|
| 4096（默认） | 通用（NTFS 默认分配单元大小），`FileStream` 默认值 |
| 8192 | 大文件顺序读写 |
| 16384+ | 极高吞吐（收益递减） |
| 1024 | 小文件、低频（需显式指定） |

### ArrayPool 复用缓冲区

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    int bytesRead;
    while ((bytesRead = await fs.ReadAsync(buffer.AsMemory(0, buffer.Length))) > 0)
    {
        ReadOnlySpan<byte> data = buffer.AsSpan(0, bytesRead);  // 零堆分配切片
        ProcessChunk(data);
    }
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

> [!tip] Span vs Memory
> `Span<T>` / `ReadOnlySpan<T>` 用于栈上操作（零堆分配），但不能跨越 await 边界；`Memory<T>` / `ReadOnlyMemory<T>` 可安全用于异步。

## 常用 API 速查

```csharp
// 一次性读
string text = await File.ReadAllTextAsync("small.txt");
string[] lines = await File.ReadAllLinesAsync("small.txt");
byte[] bytes = await File.ReadAllBytesAsync("image.png");

// 一次性写
await File.WriteAllTextAsync("output.txt", content);
await File.WriteAllLinesAsync("output.txt", lines);
await File.WriteAllBytesAsync("output.bin", bytes);

// 追加
await File.AppendAllTextAsync("log.txt", $"{DateTime.Now}: message\n");

// 流式（注意：以下便捷方法不传 FileOptions.Asynchronous，不支持真正的异步 I/O）
using var reader = File.OpenText("file.txt");         // StreamReader（同步底层）
using var writer = File.CreateText("file.txt");       // StreamWriter（同步底层）
using var stream = File.OpenRead("file.bin");         // FileStream 只读（同步底层）
using var stream = File.OpenWrite("file.bin");        // FileStream 只写（同步底层）
// 需要真正的异步 I/O 时，手动 new FileStream(... , FileOptions.Asynchronous)
```

## 决策速查

```
文件大小判断：
├── < 10 MB    → File.ReadAllTextAsync / File.WriteAllTextAsync
├── 10-100 MB  → StreamReader.ReadLineAsync（可选手动 FileStream + useAsync）
└── > 100 MB   → 建议 FileStream + ReadAsync + ArrayPool（根据服务器内存可放宽阈值）

I/O 类型：
├── 文本格式   → StreamReader / StreamWriter
├── 二进制格式 → FileStream 直接读写 byte[]
├── JSON       → JsonSerializer.SerializeAsync
└── 日志追加   → File.AppendAllTextAsync

性能要求：
├── 低延迟 UI  → 全部异步
├── 高并发服务 → FileOptions.Asynchronous + ArrayPool
└── 批处理     → 同步 + 大缓冲区可能更优
```

## 常见错误

| 错误 | 后果 | 修复 |
|------|------|------|
| `new FileStream(...)` 不传 `useAsync: true` | 伪异步，线程池阻塞 | 显式传 `FileOptions.Asynchronous` |
| 大文件用 `ReadAllText` | `OutOfMemoryException` | 使用 `StreamReader` 逐行读 |
| `await` 后忘记 `using` / `Dispose` | 文件句柄泄漏 | 使用 `await using var` |
| 同步上下文中 `.Result` / `.Wait()` | 死锁 | 全程 `async/await` |
| 默认 UTF-8 with BOM | Linux/Web 兼容问题 | `new UTF8Encoding(false)` |

## 参见

- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib/os/shutil 的分层架构与 C# File/FileInfo 的对比
