---
title: "CSharp序列化IO"
tags:
  - csharp
  - serialization
  - xml
  - file-io
  - stream
created: 2026-06-02
---

# CSharp序列化IO

C# 文件 I/O、Stream 体系、XML 操作与序列化/反序列化，以及字节编码转换。

## 核心概念

- **Stream 抽象**：所有 I/O 的基类；`Read/Write/Seek/Flush`；`Position` 使用前务必重置为 0
- **File vs FileInfo**：File 静态方法（便捷）、FileInfo 实例方法（多次操作同一文件）
- **XmlSerializer**：将对象序列化为 XML；仅序列化 public 成员、不支持 Dictionary、需要无参构造函数
- **IXmlSerializable**：自定义序列化规则，可处理 Dictionary 等特殊类型
- **BitConverter / Encoding**：基础类型与 byte[] 互转；string 编码用 UTF8（最通用）

## 常见陷阱

1. Stream 的 Position 未重置导致读写错误位置
2. 反序列化时 List 默认值不会被清空（追加而非替换）
3. 存储路径选择：`PersistentDataPath` 是唯一可读可写且打包后存在的路径
4. `File.ReadAllText` 适合小文件；大文件用 `StreamReader` 逐行读

## 与已有页面关联

- [[CSharp文件IO]] — File/FileInfo/Directory 的文件系统操作
- [[CSharp进程管理]] — Process 的标准输入输出流重定向
- [[CSharp集合框架]] — 集合的 XML 序列化与 Dictionary 的自定义序列化

## 来源

- [[csharp-serialization-摘要]]
