---
title: "CSharp委托特性"
tags:
  - csharp
  - delegate
  - event
  - attribute
  - reflection
type: concept
updated: 2026-06-02
---

# CSharp委托特性

委托（Delegate）、事件（Event）、反射（Reflection）、特性（Attribute）、参数修饰符（ref/out/in）五大 C# 核心机制。

## 核心概念

- **委托**：本质是 `MulticastDelegate` 类，支持多播（+=/-=）；内置 `Action`（无返回）和 `Func`（有返回）
- **委托 vs 接口性能**：在现代 .NET 中差异微乎其微（委托 ~1.2x, 接口 ~1.1x）。选择应基于设计意图：单函数契约用委托，多方法行为契约用接口
- **事件**：委托的封装，禁止外部 `=` 赋值和 `Invoke` 调用，只能 `+=`/`-=`；实现发布-订阅模式
- **反射**：运行时获取类型信息（`Type`、`MethodInfo`、`PropertyInfo`）；程序集包含清单、类型元数据、MSIL、资源四个部分
- **Attribute**：运行时元数据标记，三步走：声明（继承 `Attribute`）→ 应用 → 反射读取；本身不自动执行
- **ref/out/in**：ref 双向可读写；out 纯输出；in 只读引用（避免大 struct 复制）

## 面试高频

1. 委托和事件的区别？→ 事件禁止外部赋值和调用
2. abstract vs interface？→ 抽象类可有实现/字段，接口只能声明
3. const vs readonly？→ const 编译时常量；readonly 运行时常量（构造函数赋值）
4. 反射的性能？→ 慢于直接调用，避免在热路径使用
5. 委托 vs 接口如何选？→ 单函数用委托（简洁），多方法用接口（聚合内聚）

## 与已有页面关联

- [[CSharp值类型性能]] — ref/in 修饰符对大 struct 的传递优化
- [[CSharp内存GC]] — Lambda 闭包的 GC 问题（匿名类生成）
- [[依赖注入]] — 反射驱动的 DI 容器原理
- [[CSharp异步模型]] — 事件处理中的 async void

## 来源

- [[csharp-delegates-attributes-摘要]]
