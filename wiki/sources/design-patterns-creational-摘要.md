---
title: "创建型设计模式 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [design-patterns, creational, csharp, factory-method, abstract-factory, builder, prototype, singleton]
source: "raw/cs/design-patterns/creational-patterns.md"
---

# 创建型设计模式 — 摘要

来源：`raw/cs/design-patterns/creational-patterns.md`

## 概述

五种 GoF 创建型模式的完整指南：工厂方法、抽象工厂、生成器、原型、单例。每个模式包含 UML 类图（Mermaid）、完整 C# 代码示例、参与者分析、适用场景（含何时不使用）、变体、常见反模式、以及与其他模式的关系。末尾提供模式选择决策树。

## 各模式要点

- **工厂方法**：子类决定实例化哪个类——产品必须共享公共接口。Creator 的 `CreateProduct()` 是抽象工厂方法，`PlanDelivery()` 本身是模板方法。与 DI 结合时可用 `Func<T>` 委托替代传统工厂类。**避坑**：工厂返回具体类型、上帝工厂、滥用继承导致子类爆炸

- **抽象工厂**：创建一族相关产品（如跨平台 UI 组件），保证风格一致。通常由一组工厂方法构成。与生成器的区别——抽象工厂**立即返回**产品，生成器**允许额外构造步骤**。**避坑**：工厂接口膨胀、产品族泄露、强行组合不相关产品

- **生成器**：避免"重叠构造函数"。现代 C# 推荐 Fluent Builder（链式调用），可选 Director 封装步骤序列。可与组合模式结合构造复杂树。C# 中 `record with`、命名参数可部分替代。**避坑**：Builder 与 Product 强绑定、忘记调用 `Build()`、Builder 过大

- **原型**：复制构造函数实现深拷贝——`Clone()` 返回独立副本。`ICloneable` 存在语义模糊问题（浅/深不明确）。JSON 序列化提供通用深拷贝但无法处理循环引用。原型注册表用于缓存预配置原型。**避坑**：浅克隆意外共享引用、循环引用栈溢出、`ICloneable` 语义模糊

- **单例**：六种 C# 实现演进——从简单懒加载（线程不安全）→ 双重检查锁定 → 静态构造函数（CLR 保证）→ `Lazy<T>`（推荐）→ 泛型单例基类。核心建议：**优先使用 DI 容器的 `AddSingleton<T>()`** 而非 GoF 单例。**避坑**：全局状态滥用、测试困难、并发瓶颈、子类化单例

## 关键关系

- 抽象工厂由一组工厂方法构成
- 生成器、抽象工厂、原型都可与单例组合
- 所有创建型模式在现代 C# 中的首选替代是 [[concepts/依赖注入|DI 容器]]
- 工厂方法本身是模板方法模式的一种特殊形式

## 关联页面

- [[concepts/设计模式-创建型|设计模式 — 创建型]]
- [[concepts/设计模式-结构型|设计模式 — 结构型]]
- [[concepts/设计模式-行为型|设计模式 — 行为型]]
