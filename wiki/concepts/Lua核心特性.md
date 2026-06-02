---
title: "Lua 核心特性"
type: concept
updated: 2026-06-02
tags: [lua, table, closure, metatable, coroutine, gc]
aliases: [Lua核心特性, Lua底层, Lua原理]
---

# Lua 核心特性

Lua 作为轻量级嵌入式脚本语言，其简洁语法背后隐藏着精巧的运行时设计。理解 Table 结构、闭包机制、元表系统和 GC 策略是高效使用 Lua 的前提。

## Table：一体两段

Lua table 是语言中唯一的数据结构，内部由**数组段**（array part）和**哈希段**（hash part）组成。数组段存放从 1 开始的连续整数键，哈希段存放其他键。插入新键时可能触发 Rehash——重新计算最优数组段大小并重新分配空间。

> [!warning] `#` 运算符不可靠
> 对于含 nil 的 table，`#` 返回值不确定。删除元素用 `table.remove` 而非赋 nil，长度获取建议遍历 `pairs` 计数。

**三种遍历方式的选择**：数值 for 性能最高且可控方向/步长；`ipairs` 语法简洁但遇 nil 即停；`pairs` 遍历所有键但顺序不保证。优先使用数值 for。

## 闭包与 UpValue

Lua 闭包通过 **UpValue** 捕获外部局部变量。多个闭包引用同一外部变量时共享同一个 UpValue——修改互相可见。闭包创建时 UpValue 为 open 状态（指向栈值），变量离开作用域若仍被引用则提升到堆上，无引用时 closed 回收。

闭包在 Lua 中广泛用于迭代器、回调封装和模块隐藏。理解 UpValue 的共享与生命周期是调试闭包相关 bug 的关键。

## 元表：操作符重载与 OOP

元表是 Lua 实现面向对象和运算符重载的基石。核心元方法：

- **`__index`**：查找不存在的键时触发，若指向另一表则形成原型链
- **`__newindex`**：赋值不存在的键时触发，可拦截写入
- **`__call`**：将表作为函数调用
- 算术运算符（`__add` 等）和比较运算符（`__eq` 等）

基于元表的 OOP 模式：类表设置 `__index = self`，实例通过 `setmetatable({}, Class)` 建立原型链。冒号语法 `obj:method()` 自动将调用者作为 self 传入。模拟继承时通过设置元表链实现，多态需显式调用 `self.base.Method(self)`。

## 协程：协作式多任务

Lua 协程是非抢占式的，由程序主动 `yield` 让出执行权。`resume` 返回 `(ok, value)`，第一个值指示是否成功；`wrap` 返回函数调用更简洁。`yield` 和 `resume` 之间的参数可双向传递，使协程成为强大的控制流工具。

## 内存管理

Lua GC 采用**增量式三色标记-清除**算法：白色（未标记）、灰色（待扫描）、黑色（已标记）。增量式设计允许 GC 随时暂停和继续，减少停顿。弱引用表（`__mode`）用于解决循环引用问题。

## 关联页面

- [[sources/lua-core-摘要|Lua 核心特性来源摘要]]
- [[concepts/XLua热补丁|XLua 热补丁]] — Unity 中的 Lua 热更新
- [[entities/EmmyLua|EmmyLua]] — Lua 类型注解与 IDE 工具链
