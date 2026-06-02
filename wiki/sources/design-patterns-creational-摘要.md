---
title: "创建型设计模式 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [design-patterns, creational, csharp, factory, builder, singleton, prototype]
source: "raw/cs/design-patterns/creational-patterns.md"
---

# 创建型设计模式 — 摘要

来源：`raw/cs/design-patterns/creational-patterns.md`

## 概述

五种 GoF 创建型模式的完整指南：工厂方法、抽象工厂、生成器、原型、单例。所有示例使用 C# 编写，包含 UML 类图（Mermaid）和模式选择决策树。

## 要点

- **工厂方法**：子类决定实例化哪个类，产品的公共接口是必要条件。与 DI 结合时可用 `Func<T>` 委托替代传统工厂类
- **抽象工厂**：创建一族相关产品（如跨平台 UI 组件），通常由一组工厂方法构成。与生成器的区别——抽象工厂立即返回产品，生成器允许额外构造步骤
- **生成器**：六种 C# 实现方式——从传统 Director 模式到现代 Fluent Builder。C# 的 `record` 类型和 DI 容器提供了部分替代方案
- **原型**：复制构造函数实现深拷贝，`ICloneable` 存在语义模糊问题。JSON 序列化提供通用深拷贝但无法处理循环引用
- **单例**：六种 C# 实现从简单懒加载到 `Lazy<T>`，包含双重检查锁定的内存模型分析和泛型单例基类。核心建议：优先使用 DI 容器的 `AddSingleton<T>()` 而非 GoF 单例

## 关联页面

- [[concepts/设计模式-创建型|创建型设计模式]]
