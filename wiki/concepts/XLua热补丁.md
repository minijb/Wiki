---
title: "XLua 热补丁"
type: concept
updated: 2026-06-02
tags: [xlua, unity, lua, csharp, hotfix, il-injection]
aliases: [XLua热补丁, xLua热更新, XLua hotfix]
---

# XLua 热补丁

XLua 是 Unity 下最成熟的 Lua 热更新方案，核心由两部分构成：**C# ↔ Lua 双向互调**提供脚本化能力，**IL 注入热补丁**实现运行时 C# 函数替换。

## 双向互调映射

### C# → Lua

C# 读取 Lua 数据有四种映射方式，选择取决于同步需求和性能要求：

| 映射方式 | 机制 | 推荐场景 |
|----------|------|----------|
| `class`/`struct` | 值拷贝，字段名对应 | 一次性读取，不关心后续变化 |
| `interface` + `[CSharpCallLua]` | 引用拷贝，get/set 实时同步 | 需要双向同步的首选 |
| `Dictionary<,>`/`List<>` | 值拷贝 | 简单 key-value 或数组 |
| `LuaTable` | 引用，无代码生成 | 原型阶段，正式发布不推荐（慢 ~10x） |

函数映射优先使用 Delegate + `[CSharpCallLua]`。多返回值通过 `out`/`ref` 参数从左往右对应 Lua 的多返回值。

### Lua → C#

所有 C# 类型挂载在 `CS` 全局命名空间下。关键差异：
- Lua 无 `new` 关键字，直接 `CS.XXX()` 构造
- 成员方法必须用**冒号**调用
- `ref` 参数需传入占位值，`out` 不需要
- 泛型容器通过 `CS.System.Collections.Generic.List(CS.System.String)` 获取类型实例
- Dictionary 访问不能用 `dic["key"]`，必须用 `get_Item`/`set_Item`

## 性能陷阱与优化

> [!warning] 串联 `.` 访问代价高
> 每次 `obj.transform.position` 涉及：userdata → id 转换 → ObjectTranslator 字典查找 → 返回 C# 对象。每层 `.` 触发一次查找。

**核心优化原则**：
1. **持有对象引用**：避免临时 userdata 被 GC 后反复创建
2. **使用静态工具方法**：减少 object 查找链路，如 `LuaUtil.SetPos(obj, x, y, z)`
3. **避免传 `Vector3`/`Quaternion`**：用三个 float 替代，省去 3 次 push + table 分配 + 3 次表插入
4. **标注 `[LuaCallCSharp]`/`[CSharpCallLua]`**：生成适配代码替代反射，性能差异可达 10 倍
5. **`[GCOptimize]`**：值类型传递不装箱，零 GC alloc

## 热补丁原理

热补丁分为三个步骤，分别在编译期和运行期完成：

### 1. Generate Code（编译期）

扫描所有 `[Hotfix]` 标注的类，生成 `DelegateBridge` 类。每个 `DelegateBridge` 实例持有一个 `luaReference` 指向 Lua 函数，通过 `__Gen_Delegate_Imp` 桥接到 Lua 侧。

### 2. Hotfix Inject（编译期 IL 注入）

使用 **Mono.Cecil** 修改程序集 IL。为每个 `[Hotfix]` 方法注入一个 `static DelegateBridge` 字段和方法体的条件分支：

```
if (DelegateBridge != null)
    执行 Lua 补丁函数
else
    执行原始 C# 逻辑
```

### 3. xlua.hotfix()（运行期）

在 Lua 中调用 `xlua.hotfix(CS.XXX, "MethodName", luaFunction)` 将 Lua 函数绑定到对应的 `DelegateBridge`。此后每次调用该方法，IL 注入的分支便会路由到 Lua 实现。

> [!tip] 热补丁不是"替换"而是"劫持"
> 原始 C# 代码仍存在于程序集中。当 `DelegateBridge` 为 null 时自动回退到原始逻辑。构造函数和析构函数的补丁特殊——先执行原有逻辑再调用 Lua。

## 热补丁覆盖范围

| 成员类型 | method_name | 备注 |
|----------|-------------|------|
| 普通方法 | 方法名 | 支持重载，成员方法多一个 self 参数 |
| 构造函数 | `".ctor"` | 先执行原逻辑再调 Lua |
| 属性 getter | `get_属性名` | — |
| 属性 setter | `set_属性名` | — |
| 索引器 | `get_Item` / `set_Item` | — |
| 事件 | `add_事件名` / `remove_事件名` | — |
| 析构函数 | `Finalize` | 先调 Lua 再继续原逻辑 |
| 协程 | 通过 `util.cs_generator` 包装 | — |
| 泛型类 | 需按实例化类型分别打补丁 | `CS.GenericClass(CS.System.Double)` |

## 关联页面

- [[sources/xlua-hotfix-摘要|XLua 互调与热补丁来源摘要]]
- [[concepts/Lua核心特性|Lua 核心特性]] — Lua 语言底层机制
- [[entities/EmmyLua|EmmyLua]] — Lua 类型注解工具链
