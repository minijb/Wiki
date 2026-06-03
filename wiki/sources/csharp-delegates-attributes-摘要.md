---
title: "CSharp委托特性-摘要"
source: "raw/cs/languages/csharp-delegates-attributes.md"
type: source-summary
updated: 2026-06-02
tags:
  - csharp
  - delegate
  - event
  - attribute
---

# CSharp委托特性-摘要

> 源文件: [[CSharp委托特性]]

## 核心内容

覆盖委托（Delegate）、事件（Event）、反射（Reflection）、特性（Attribute）、参数修饰符（ref/out/in）五大话题：委托本质是 MulticastDelegate 类、Event 是对委托的封装（禁止外部赋值和调用）、反射的核心类型和基本操作、自定义 Attribute 的三步骤（声明→应用→反射读取）、ref/out/in 的语义与使用场景、泛型约束、闭包面试要点、委托 vs 接口性能对比。

## 关键要点

1. **委托本质**：是一个类，支持多播；Action 无返回，Func 有返回
2. **委托 vs 接口性能**：现代 .NET 中差异微乎其微（委托 ~1.2x, 接口 ~1.1x），选择应基于设计意图
3. **Event vs Delegate**：Event 禁止外部 `=` 和 `Invoke`，只能 `+=`/`-=`，防止意外覆盖
4. **反射四件套**：程序集清单、类型元数据、MSIL 代码、资源
5. **Attribute 三步**：声明（继承 Attribute）→ 应用 → 反射读取；本身不自动执行
6. **ref vs out vs in**：ref 可读写（需初始化）；out 纯输出（方法内赋值）；in 只读引用（避免大 struct 复制）

## 相关页面

- [[CSharp委托特性]]
- [[CSharp值类型性能]]
- [[依赖注入]]
