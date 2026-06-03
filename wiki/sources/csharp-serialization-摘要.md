---
title: "CSharp序列化IO-摘要"
source: "raw/cs/languages/csharp-serialization.md"
type: source-summary
updated: 2026-06-02
tags:
  - csharp
  - serialization
  - xml
  - json
---

# CSharp序列化IO-摘要

> 源文件: [[CSharp序列化IO]]

## 核心内容

覆盖 C# I/O 体系：Stream 抽象类及其子类（FileStream/MemoryStream/NetworkStream）、同步/异步读写操作对比、File/FileInfo/Directory 文件系统操作、BitConverter/Encoding 字节转换、XML DOM 操作（XmlDocument 的读取/创建/修改）、XmlSerializer 序列化/反序列化（含特性标记和限制）、IXmlSerializable 自定义序列化、可序列化 Dictionary 实现。

**v2 新增**：System.Text.Json 现代 JSON 序列化（快 5-10x，源生成器 AOT 友好，零分配选项），与 XmlSerializer 的全维度对比。

## 关键要点

1. **Stream Position 陷阱**：多次使用前务必重置为 0，否则读写位置错误
2. **System.Text.Json**：快 5-10x（源生成器），原生支持 Dictionary，AOT 友好 — XmlSerializer 的现代替代
3. **XmlSerializer 限制**：仅 public 成员、不支持 Dictionary、需要无参构造函数
4. **Encoding 选择**：UTF8 最通用；ASCII 仅英文；Default 平台不安全
5. **存储路径**：PersistentDataPath 是唯一可读可写且打包后存在的路径

## 相关页面

- [[CSharp文件IO]]
- [[CSharp序列化IO]]
- [[CSharp进程管理]]
- [[CSharp集合框架]]
