---
title: "CSharp序列化IO"
tags:
  - csharp
  - serialization
  - xml
  - file-io
  - stream
  - json
type: concept
updated: 2026-06-02
---

# CSharp序列化IO

C# 文件 I/O、Stream 体系、XML 操作与序列化/反序列化，以及 `System.Text.Json` 作为现代高性能替代。

## 核心概念

- **Stream 抽象**：所有 I/O 的基类；`Read/Write/Seek/Flush`；`Position` 使用前务必重置为 0
- **File vs FileInfo**：File 静态方法（便捷）、FileInfo 实例方法（多次操作同一文件）
- **XmlSerializer**：将对象序列化为 XML；仅序列化 public 成员、不支持 Dictionary、需要无参构造函数
- **System.Text.Json**（.NET Core 3.0+）：XmlSerializer 的现代替代，快 5-10x（源生成器），零分配选项（Utf8JsonWriter），AOT 友好，原生支持 Dictionary
- **IXmlSerializable**：自定义序列化规则，可处理 Dictionary 等特殊类型
- **BitConverter / Encoding**：基础类型与 byte[] 互转；string 编码用 UTF8（最通用）

## XmlSerializer vs System.Text.Json

| 维度 | XmlSerializer | System.Text.Json |
|------|--------------|------------------|
| 速度 | 较慢（反射 + 动态 IL 生成） | 快 5-10x（源生成器） |
| 分配 | 每次缓存序列化程序集 | 零分配选项（Utf8JsonWriter） |
| 格式 | XML（冗长） | JSON（紧凑） |
| Dictionary 支持 | 需 IXmlSerializable | 原生支持 |
| 源生成器（.NET 6+） | 不支持 | ✅ AOT 友好 |

## 常见陷阱

1. Stream 的 Position 未重置导致读写错误位置
2. 反序列化时 List 默认值不会被清空（追加而非替换）
3. 存储路径选择：`PersistentDataPath` 是唯一可读可写且打包后存在的路径
4. `File.ReadAllText` 适合小文件；大文件用 `StreamReader` 逐行读

## 与已有页面关联

- [[CSharp文件IO]] — File/FileInfo/Directory 的文件系统操作
- [[CSharp进程管理]] — Process 的标准输入输出流重定向
- [[CSharp集合框架]] — 集合的 XML/JSON 序列化与 Dictionary 的自定义序列化

## 来源

- [[csharp-serialization-摘要]]
