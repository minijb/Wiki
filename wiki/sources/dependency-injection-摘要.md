---
title: "依赖注入-摘要"
source: "raw/cs/architecture/dependency-injection.md"
type: source-summary
created: 2026-06-02
---

# 依赖注入-摘要

> 源文件: [[依赖注入]]

## 核心内容

从 RPG 游戏武器系统出发，循序渐进讲解依赖注入的动机、定义、三种注入方式（Setter/构造/依赖获取）、反射驱动的 DI 容器原理与简易实现、Transient/Singleton/Scoped 三种生命周期及陷阱、DI 与 SOLID 原则的关系、DI 与 Strategy/Factory 设计模式的关系、ASP.NET Core 和 Unity 的实际应用、常见面试题解答。

## 关键要点

1. **DI 定义**：将依赖的实例化从客户类移出，由外部注入；是 IoC 的实现方式
2. **构造注入首选**：依赖显式、编译时强制、创建后不可变
3. **Service Locator 是反模式**：隐藏依赖、运行时错误、难测试
4. **反射驱动的容器**：通过类型名+反射实现配置化实例化，不改代码支持未来扩展
5. **生命周期陷阱**：Singleton 中的 Transient 依赖被提升为事实 Singleton；不能将 Scoped 注入 Singleton

## 相关页面

- [[依赖注入]]
- [[CSharp值类型性能]]
- [[CSharp异步模型]]
