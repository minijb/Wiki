---
title: "行为型设计模式 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [design-patterns, behavioral, csharp, chain-of-responsibility, command, iterator, mediator, memento, observer, state, strategy, template-method, visitor]
source: "raw/cs/design-patterns/behavioral-patterns.md"
---

# 行为型设计模式 — 摘要

来源：`raw/cs/design-patterns/behavioral-patterns.md`

## 概述

十种 GoF 行为型模式的完整指南，涵盖责任链、命令、迭代器、中介者、备忘录、观察者、状态、策略、模板方法、访问者。所有示例使用 C# 编写，包含 UML 类图、模式对比和组合模式关系图。

## 要点

- **责任链**：请求沿链传递直到被处理。ASP.NET Core 中间件管道是经典应用——每个中间件可处理或调用 `_next()`。C# 实现支持流畅链式组装
- **命令**：将请求封装为对象，支持撤销/重做、任务队列。结合备忘录模式实现完整撤销历史。与策略的区别——命令封装请求及其参数，策略封装可互换算法
- **迭代器**：C# 语言内置支持——`IEnumerable<T>` + `yield return`。编译器自动生成状态机，无需手写迭代器类
- **中介者**：组件通过中介者通信，解除多对多耦合。登录对话框示例展示 `CheckBox`、`TextBox`、`Button` 通过 `LoginDialog` 协调
- **备忘录**：原发器自行创建快照，备忘录对外不透明。结合命令模式实现撤销——命令执行前保存备忘录，撤销时恢复
- **观察者**：C# 原生通过 `event` 关键字支持。`IObservable<T>` / `IObserver<T>` 提供标准化的可订阅流。优先用 `event` 替代手动维护观察者列表
- **状态**：有限状态机实现——每个状态封装行为和转换规则。与策略的区别——状态知道自己何时转换到下一状态，策略由外部切换
- **策略**：算法族的可互换封装。支付策略示例展示运行时切换。与命令的核心区别在于意图而非结构
- **模板方法**：好莱坞原则——"Don't call us, we'll call you"。抽象步骤（必须实现）、默认步骤、钩子（可选扩展点）三层结构
- **访问者**：双分派机制——`Accept(visitor)` → `visitor.VisitXxx(this)`。适用于元素层次稳定但需频繁添加新操作的场景。XML 导出和面积计算两个访问者示例

## 关联页面

- [[concepts/设计模式-行为型|行为型设计模式]]
