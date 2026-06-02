---
title: "结构型设计模式 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [design-patterns, structural, csharp, adapter, bridge, composite, decorator, facade, flyweight, proxy]
source: "raw/cs/design-patterns/structural-patterns.md"
---

# 结构型设计模式 — 摘要

来源：`raw/cs/design-patterns/structural-patterns.md`

## 概述

七种 GoF 结构型模式的完整指南，涵盖适配器、桥接、组合、装饰、外观、享元、代理。所有示例使用 C# 编写，包含 UML 类图和模式关系图。

## 要点

- **适配器**：对象适配器（组合）是 C# 唯一可行方式——将 `SquarePeg` 适配为 `IRoundPeg`。`System.Data.Common` 中的 `DbDataAdapter` 是 .NET 中的经典适配器
- **桥接**：将抽象（遥控器）与实现（设备）分离。`Stream` + `TextReader`/`TextWriter` 是 .NET 桥接的经典示例
- **组合**：递归组合的树形结构，叶节点与组合节点共享接口 `IGraphic`。.NET 的 `System.Xml.XmlNode` 是组合模式的直接体现
- **装饰**：与代理的结构几乎相同但意图不同——装饰添加行为，代理控制访问。ASP.NET Core 中间件管道是装饰的经典应用；`Stream` 的 `CryptoStream` / `GZipStream` 也是装饰
- **外观**：为复杂子系统提供简化入口。`VideoConverter` 封装多个编码/解码类。外观是单向的（客户端→子系统），中介者是双向的（组件间通信）
- **享元**：内在/外在状态分离，通过工厂缓存共享对象。C# 中 `string.Intern()` 和 `Brushes.Red` 是享元的内置实现
- **代理**：六种变体——虚拟代理（`Lazy<T>`）、保护代理、远程代理（gRPC stub）、日志代理、缓存代理、智能引用

## 关联页面

- [[concepts/设计模式-结构型|结构型设计模式]]
