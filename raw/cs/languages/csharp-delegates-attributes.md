---
title: "C# 委托、事件、Attribute与参数修饰符深度解析"
tags:
  - csharp
  - delegate
  - event
  - attribute
  - reflection
type: language
created: 2026-06-02
source_files:
  - drafts/My_Vault/files/C sharpe Event, Delegates, Action.md
  - drafts/My_Vault/files/C Sharp Attribute.md
  - drafts/My_Vault/files/C sharp 中 in， out 和 ref 关键字.md
  - drafts/My_Vault/files/面试实录 -- Csharp.md
  - drafts/My_Vault/files/面试 - csharp.md
---

# C# 委托、事件、Attribute与参数修饰符深度解析

## 1. 委托（Delegate）

### 1.1 本质：委托是一个类

```csharp
// 声明委托类型 —— 本质上定义了一个类
public delegate int Operation(int a, int b);

// 编译器生成（简化等效）：
// public sealed class Operation : System.MulticastDelegate
// {
//     public Operation(object target, IntPtr methodPtr);
//     public virtual int Invoke(int a, int b);
//     public virtual IAsyncResult BeginInvoke(int a, int b, AsyncCallback callback, object state);
//     public virtual int EndInvoke(IAsyncResult result);
// }
```

委托是**多播**的：一个委托实例可以持有多个方法的引用，调用时依次执行。

### 1.2 委托的使用

```csharp
// 声明委托类型
public delegate void MyDelegate(string message);

// 创建委托实例
MyDelegate del = Console.WriteLine;     // 方法组转换
del += msg => Debug.Log(msg);           // Lambda 表达式
del += delegate(string msg) { };        // 匿名方法（旧语法）

// 调用
del?.Invoke("Hello");

// 移除
del -= Console.WriteLine;
```

### 1.3 内置泛型委托

```csharp
// Action —— 无返回值，最多 16 个参数
Action action = () => Console.WriteLine("Done");
Action<int, string> action2 = (n, s) => Console.WriteLine($"{n}: {s}");

// Func —— 有返回值，最后一个泛型参数是返回类型
Func<int> func0 = () => 42;
Func<int, int, int> add = (a, b) => a + b;  // 输入 int,int → 返回 int
Func<int, bool> isPositive = x => x > 0;

// Predicate —— 返回 bool，等价于 Func<T, bool>
Predicate<int> isEven = x => x % 2 == 0;

// EventHandler —— 标准事件委托
public event EventHandler<MyEventArgs> SomethingHappened;
```

### 1.4 委托 vs 接口

相同点：都定义了一组行为契约。
不同点：
- 委托定义**单个**可调用契约（一个函数签名）
- 接口定义**一组**行为契约（多个方法）
- 委托天然支持多播（+=/-=）

```csharp
// 用接口定义策略
interface IComparator { int Compare(object a, object b); }

// 用委托定义策略（更简洁）
Func<object, object, int> comparator;
```

---

## 2. 事件（Event）

### 2.1 Event 是委托的封装

事件在委托基础上增加了**访问控制**：

```csharp
// 委托 —— 任何人可以调用、赋值
public Action OnCompleted;
OnCompleted = SomeMethod;   // 可行！覆盖所有订阅
OnCompleted?.Invoke();      // 任何人可调用

// 事件 —— 仅声明类内部可调用；外部只能 += / -=
public event Action OnCompleted;
OnCompleted += SomeMethod;  // ✅ 外部可订阅
OnCompleted -= SomeMethod;  // ✅ 外部可取消订阅
OnCompleted?.Invoke();      // ✅ 仅声明类内部可调用
// OnCompleted = SomeMethod; // ❌ 外部不可赋值（编译错误）
```

### 2.2 事件的使用模式

```csharp
public class Player
{
    // 方式1：使用 EventHandler<T>
    public event EventHandler<HealthChangedArgs> HealthChanged;

    // 方式2：自定义委托
    public delegate void DamageDelegate(float amount);
    public event DamageDelegate OnDamaged;

    // 方式3：使用 Action
    public event Action<int, string> OnLevelUp;

    // 触发事件的标准模式
    protected virtual void OnHealthChanged(int oldHealth, int newHealth)
    {
        HealthChanged?.Invoke(this, new HealthChangedArgs(oldHealth, newHealth));
    }
}

public class HealthChangedArgs : EventArgs
{
    public int OldHealth { get; }
    public int NewHealth { get; }
    public HealthChangedArgs(int old, int @new) { OldHealth = old; NewHealth = @new; }
}

// 订阅
player.HealthChanged += (sender, args) =>
{
    Debug.Log($"HP: {args.OldHealth} → {args.NewHealth}");
};
```

### 2.3 事件的核心价值

1. **防止意外覆盖**：外部不能用 `=` 清掉别人订阅的方法
2. **封装触发逻辑**：只有声明事件的类才能触发
3. **实现发布-订阅模式**：解耦发送者和接收者

---

## 3. 反射（Reflection）

### 3.1 反射的核心类型

| 类型 | 描述 |
|------|------|
| `Assembly` | 程序集 |
| `Type` | 类型信息（类、结构、接口） |
| `MethodInfo` | 方法信息 |
| `PropertyInfo` | 属性信息 |
| `FieldInfo` | 字段信息 |
| `ConstructorInfo` | 构造函数信息 |
| `MemberInfo` | 所有成员的基类 |

### 3.2 基本操作

```csharp
// 获取类型
Type t = typeof(MyClass);
Type t2 = obj.GetType();
Type t3 = Type.GetType("Namespace.MyClass, AssemblyName");

// 创建实例
object instance = Activator.CreateInstance(t);

// 获取并调用方法
MethodInfo method = t.GetMethod("MyMethod");
method.Invoke(instance, new object[] { arg1, arg2 });

// 获取并设置属性
PropertyInfo prop = t.GetProperty("Name");
prop.SetValue(instance, "Alice");
string name = (string)prop.GetValue(instance);

// 获取自定义特性
object[] attrs = t.GetCustomAttributes(typeof(MyAttribute), inherit: true);
```

### 3.3 程序集元数据

程序集包含四个部分：
1. **程序集元数据（清单）**：名称、版本、文化、公钥、文件列表、引用程序集列表
2. **类型元数据**：定义了哪些类、每个类的成员、方法签名等
3. **MSIL 代码**：编译后的中间语言
4. **资源**：图片、图标、字符串等嵌入资源

---

## 4. Attribute（特性）

### 4.1 概念

Attribute 是**在运行时传递信息的标记**，附加到类型、方法、属性、参数等元素上。本身不执行逻辑，通过**反射读取**来实现功能。

```csharp
// 常见内置特性
[Serializable]
[Obsolete("Use NewMethod instead")]
[Conditional("DEBUG")]
[DebuggerDisplay("Name = {Name}")]
public class MyClass { }
```

### 4.2 自定义特性

```csharp
// 步骤1：声明特性 —— 继承 System.Attribute
[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Method | AttributeTargets.Property,
    AllowMultiple = true,     // 同一元素可多次应用
    Inherited = true)]        // 派生类是否继承
public class BugFixAttribute : Attribute
{
    public int BugId { get; }
    public string Developer { get; }
    public string LastReview { get; }
    public string? Message { get; set; }

    public BugFixAttribute(int bugId, string developer, string lastReview)
    {
        BugId = bugId;
        Developer = developer;
        LastReview = lastReview;
    }
}

// 步骤2：应用特性
[BugFix(101, "Alice", "2024-01-15", Message = "Fixed null reference")]
public class CustomerService
{
    [BugFix(102, "Bob", "2024-02-20")]
    public void CalculateDiscount() { }
}

// 步骤3：通过反射读取特性
MemberInfo member = typeof(CustomerService);
foreach (var attr in member.GetCustomAttributes<BugFixAttribute>())
{
    Console.WriteLine($"Bug #{attr.BugId} fixed by {attr.Developer}: {attr.Message}");
}
```

### 4.3 AttributeUsage 参数

| 参数 | 说明 |
|------|------|
| `AttributeTargets` | 可应用的目标（Class、Method、Property 等） |
| `AllowMultiple` | 同一元素是否允许多个实例 |
| `Inherited` | 派生类是否自动继承 |

---

## 5. ref / out / in 参数修饰符

### 5.1 语义对比

| 修饰符 | 传入 | 传出 | 必须初始化 | 使用场景 |
|--------|------|------|-----------|----------|
| （无） | 值复制 | 不影响原值 | — | 默认按值传递 |
| `ref` | 引用传入 | 可读写 | 调用**前**必须初始化 | 既要读又要改 |
| `out` | 不读（方法必须赋值） | 传出 | 调用**后**方法必须赋值 | 纯返回值（TryParse 模式） |
| `in` | 只读引用 | 不可写 | 调用前必须初始化 | 只读传递大 struct |

### 5.2 代码示例

```csharp
// ref —— 双向传递
void Modify(ref int x)
{
    x = x * 2;  // 读取 + 修改
}
int a = 5;
Modify(ref a);  // a = 10

// out —— 纯输出
bool TryParse(string s, out int result)
{
    if (int.TryParse(s, out result))  // 方法内必须给 result 赋值
        return true;
    result = 0;
    return false;
}

// in —— 只读引用（避免大 struct 复制）
double Distance(in Vector3 a, in Vector3 b)
{
    // a.X = 5;  // 编译错误！in 参数不可修改
    return Math.Sqrt((a.X - b.X) * (a.X - b.X) + ...);
}

Vector3 v1 = new(0, 0, 0), v2 = new(3, 4, 0);
double d = Distance(in v1, in v2);  // 按引用传递，无复制
```

### 5.3 值类型 vs 引用类型

```csharp
// 值类型（struct）—— 在栈上内联存储
struct Point { public int X, Y; }

// 引用类型（class）—— 在堆上分配
class Person { public string Name; }

// 常见的值类型：struct, int, float, bool, enum, ValueTuple
// 常见的引用类型：class, interface, delegate, object, string, array
```

---

## 6. 面试要点

### 6.1 委托相关
1. **委托本质**：是一个类（MulticastDelegate），持有方法列表，支持多播
2. **Event vs Delegate**：Event 是委托的封装，禁止外部赋值和调用，只能 +=/-=
3. **Action vs Func**：Action 无返回值，Func 有返回值；都是内置泛型委托
4. **委托与接口的联系**：都定义行为契约；委托是单一函数契约，接口是多方法契约

### 6.2 反射相关
1. **反射的作用**：运行时获取类型信息、创建实例、调用方法、访问属性字段
2. **程序集四部分**：清单（元数据）、类型元数据、MSIL 代码、资源
3. **性能问题**：反射慢于直接调用；避免在热路径频繁使用


## 7. 泛型约束（面试补充）

```csharp
// 每个泛型参数至多一个主要约束
void Method<T>(T param) where T : class      // T 必须是引用类型
void Method<T>(T param) where T : struct     // T 必须是值类型
void Method<T>(T param) where T : BaseClass  // T 必须继承自 BaseClass
void Method<T>(T param) where T : new()      // T 必须有无参构造函数

// 可以有无限个次要约束（接口）
void Method<T>(T param) where T : IDisposable, IComparable<T>

// 多重约束组合
void Method<T>(T param) where T : class, IDisposable, new()
```

## 8. 闭包面试要点

1. Lambda 捕获外部变量 → 编译器生成匿名类（堆分配）
2. 每次进入含 Lambda 的方法，闭包对象重新创建
3. `foreach` 循环中捕获循环变量：C# 5.0+ 已修复（每次迭代独立变量）
4. 闭包的 GC 影响：高频调用路径避免匿名函数捕获变量
### 6.3 Attribute 相关
1. **三步走**：声明 → 应用 → 反射读取
2. **不自动执行**：特性只是元数据标记，需通过反射主动读取
3. **常见内置特性**：`[Serializable]`、`[Obsolete]`、`[Conditional]`

### 6.4 参数修饰符相关
1. **ref vs out**：ref 要求调用前初始化（可读），out 要求方法内赋值（纯输出）
2. **in 的用途**：避免大 struct 的按值复制开销，表达只读语义
3. **string 是引用类型**，但行为像不可变值类型

### 6.5 C# 基础
1. **密封类（sealed）**：不能被继承；用于方法时必须配合 override
2. **abstract vs interface**：抽象类可含实现/字段/构造函数，接口只能声明；接口可多继承
3. **虚方法/抽象类/接口**：C# 多态的三种实现方式
4. **隐藏方法（new）**：子类方法加 `new` 显式隐藏基类同名方法，用于弥补设计不足
5. **const vs readonly**：const 编译时常量（仅内置类型）、readonly 运行时常量（构造函数赋值）
6. **静态构造函数的执行时机**：创建第一个实例或引用任何静态成员之前，CLR 自动调用
7. **默认接口方法（C# 8.0+）**：接口可以有默认实现，减少破坏性变更
