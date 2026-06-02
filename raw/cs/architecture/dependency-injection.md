---
title: "C# 依赖注入深度解析"
tags:
  - csharp
  - dependency-injection
  - di
  - ioc
  - design-patterns
  - architecture
type: architecture
created: 2026-06-02
source_files:
  - drafts/My_Vault/02_Knowledge/依赖注入/00_依赖注入_背景.md
---

# C# 依赖注入深度解析

## 1. 从动机开始 —— 为什么需要依赖注入

### 1.1 问题场景

以 RPG 游戏为例：玩家角色持有武器攻击怪物。朴素实现：

```csharp
public class Role
{
    public int HP { get; set; }
    public string WeaponType { get; set; }  // "WoodSword", "IronSword", "MagicSword"

    public void Attack(Monster monster)
    {
        int damage = 0;
        if (WeaponType == "WoodSword")
            damage = 20;
        else if (WeaponType == "IronSword")
            damage = 50;
        else if (WeaponType == "MagicSword")
            damage = 100;

        monster.HP -= damage;
        if (monster.HP <= 0)
            Console.WriteLine("Monster died!");
    }
}
```

**问题**：
1. **违反 OCP（开闭原则）**：每增加一种新武器，必须修改 `Attack` 方法（if-else 膨胀）
2. **职责不单一**：Role 同时负责计算伤害、扣血、判断死亡
3. **紧耦合**：武器逻辑硬编码在 Role 中

### 1.2 Strategy 模式引入

```csharp
// 步骤1：定义武器接口 —— 抽象变化部分
public interface IAttackStrategy
{
    int CalcDamage();
}

// 步骤2：实现具体武器
public class WoodSword : IAttackStrategy
{
    public int CalcDamage() => 20;
}

public class IronSword : IAttackStrategy
{
    public int CalcDamage() => 50;
}

public class MagicSword : IAttackStrategy
{
    public int CalcDamage() => 100;
}

// 步骤3：Role 依赖接口而非具体类
public class Role
{
    public int HP { get; set; }
    public IAttackStrategy Weapon { get; set; }  // ← 注入点

    public void Attack(Monster monster)
    {
        int damage = Weapon.CalcDamage();
        monster.TakeDamage(damage);
    }
}

// Monster 承担自己的职责
public class Monster
{
    public int HP { get; set; }

    public void TakeDamage(int damage)
    {
        HP -= damage;
        if (HP <= 0)
            Console.WriteLine("Monster died!");
    }
}
```

**改进点**：
- 类数量增加但每个类更短、职责更清晰
- 新武器 → 新建类实现 `IAttackStrategy`，**不修改 Role 和 Monster**
- 对扩展开放，对修改关闭

### 1.3 依赖注入的定义

> **依赖注入（DI）是一个过程的称谓：将具体依赖的实例化从客户类中移出，由外部（调用方或容器）负责创建并注入。**

核心矛盾：客户类不应用 `new` 实例化具体依赖（违反 DIP），但又需要具体实例来工作。DI 解决此矛盾。

```csharp
// 注入过程：外部实例化具体武器，赋给 Role 的注入点
Role player = new Role { HP = 100 };
player.Weapon = new MagicSword();  // ← 这就是依赖注入
player.Attack(monster);
```

---

## 2. 依赖注入的三种方式

### 2.1 Setter 注入（属性注入）

```csharp
public class UserService
{
    // 注入点：公开属性
    public ILogger Logger { get; set; }
    public IUserRepository Repository { get; set; }

    public void CreateUser(string name)
    {
        Logger.Log($"Creating user: {name}");
        Repository.Save(new User(name));
    }
}

// 使用
var service = new UserService
{
    Logger = new FileLogger(),
    Repository = new SqlUserRepository()
};
service.CreateUser("Alice");
```

**优点**：灵活，可选依赖（不注入也可运行）；适合已有代码改造。
**缺点**：依赖可能被遗忘，导致运行时 NullReferenceException；可被运行时修改。

### 2.2 构造注入（Constructor Injection）

```csharp
public class UserService
{
    private readonly ILogger _logger;
    private readonly IUserRepository _repository;

    // 注入点：构造函数参数
    public UserService(ILogger logger, IUserRepository repository)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }

    public void CreateUser(string name)
    {
        _logger.Log($"Creating: {name}");
        _repository.Save(new User(name));
    }
}

// 使用
var service = new UserService(new FileLogger(), new SqlUserRepository());
```

**优点**：依赖明确、编译时强制、创建后不变（不可变性）、易于测试。
**缺点**：构造函数参数过多时难以阅读。

> **推荐**：优先使用构造注入。它使依赖显式化，对象创建后即处于可用状态。

### 2.3 依赖获取（Service Locator）

```csharp
// 全局服务定位器
public static class ServiceLocator
{
    private static readonly Dictionary<Type, object> _services = new();

    public static void Register<T>(T service) => _services[typeof(T)] = service;
    public static T Resolve<T>() => (T)_services[typeof(T)];
}

public class UserService
{
    public void CreateUser(string name)
    {
        // 主动获取依赖
        var logger = ServiceLocator.Resolve<ILogger>();
        var repo = ServiceLocator.Resolve<IUserRepository>();
        logger.Log($"Creating: {name}");
        repo.Save(new User(name));
    }
}
```

**优点**：隐藏依赖获取细节。
**缺点**：
- 依赖不显式（看方法签名不知道需要什么）
- 运行时错误（注册缺失 → 运行时才知道）
- 难以单元测试（必须配置全局状态）
- **是反模式**，现代 DI 实践中避免

---

## 3. 反射驱动的 DI 容器

### 3.1 为什么需要容器

手动注入可行但繁琐：
```csharp
var repo = new SqlUserRepository();
var logger = new FileLogger();
var notifier = new EmailNotifier();
var validator = new UserValidator();
var service = new UserService(repo, logger, notifier, validator);
// ... 每个创建点都要重复这些代码
```

DI 容器（IoC Container）自动管理依赖的创建、组装和生命周期。

### 3.2 反射：让容器"预见未来"

核心问题：`new` 只能实例化当前存在的类。如果未来新增实现，代码必须修改。反射允许**通过类名字符串实例化对象**，配合配置文件，在不改代码的情况下加载未来出现的类。

```csharp
// 反射工厂 —— 通过配置决定具体类型
public static class ReflectionFactory
{
    private static readonly Dictionary<string, Type> _registrations = new();

    static ReflectionFactory()
    {
        // 从配置文件读取类型名，通过反射获取 Type
        var config = XDocument.Load("di-config.xml");
        foreach (var reg in config.Root.Elements("register"))
        {
            string interfaceName = reg.Attribute("interface").Value;
            string className = reg.Attribute("class").Value;
            Type interfaceType = Type.GetType(interfaceName);
            Type classType = Type.GetType(className);
            _registrations[interfaceType.FullName] = classType;
        }
    }

    public static T Resolve<T>()
    {
        Type implType = _registrations[typeof(T).FullName];
        return (T)Activator.CreateInstance(implType);  // 反射创建实例
    }
}

// 配置文件 di-config.xml
// <container>
//   <register interface="ILogger" class="FileLogger, MyApp"/>
//   <register interface="IUserRepository" class="SqlUserRepository, MyApp"/>
// </container>
```

### 3.3 简易 DI 容器实现

```csharp
public enum Lifetime { Transient, Singleton, Scoped }

public class SimpleContainer
{
    private readonly Dictionary<Type, ServiceDescriptor> _registrations = new();
    private readonly Dictionary<Type, object> _singletons = new();

    public void Register<TInterface, TImpl>(Lifetime lifetime = Lifetime.Transient)
        where TImpl : TInterface
    {
        _registrations[typeof(TInterface)] = new ServiceDescriptor(
            typeof(TImpl), lifetime);
    }

    public void RegisterInstance<TInterface>(TInterface instance)
    {
        _singletons[typeof(TInterface)] = instance;
    }

    public T Resolve<T>()
    {
        return (T)Resolve(typeof(T));
    }

    private object Resolve(Type type)
    {
        if (_singletons.TryGetValue(type, out var singleton))
            return singleton;

        if (!_registrations.TryGetValue(type, out var descriptor))
            throw new InvalidOperationException($"Type {type.Name} is not registered");

        // 反射获取构造函数，递归解析依赖
        var ctor = descriptor.ImplementationType.GetConstructors()
            .OrderByDescending(c => c.GetParameters().Length)
            .First();

        var parameters = ctor.GetParameters()
            .Select(p => Resolve(p.ParameterType))
            .ToArray();

        object instance = Activator.CreateInstance(descriptor.ImplementationType, parameters);

        if (descriptor.Lifetime == Lifetime.Singleton)
            _singletons[type] = instance;

        return instance;
    }

    private record ServiceDescriptor(Type ImplementationType, Lifetime Lifetime);
}
```

---

## 4. 生命周期管理

| 生命周期 | 说明 | 典型场景 |
|----------|------|----------|
| **Transient** | 每次解析创建新实例 | 轻量无状态服务 |
| **Singleton** | 整个应用生命周期仅一个实例 | 配置、缓存、日志工厂 |
| **Scoped** | 每个作用域（如 HTTP 请求）一个实例 | 数据库上下文（DbContext） |

```csharp
// Transient —— 每次 new
container.Register<ILogger, ConsoleLogger>(Lifetime.Transient);

// Singleton —— 全局唯一
container.Register<IConfiguration, AppConfig>(Lifetime.Singleton);

// Scoped —— 每个请求一个
using (var scope = container.BeginScope())
{
    var db1 = scope.Resolve<IDbContext>();  // 同一个实例
    var db2 = scope.Resolve<IDbContext>();  // 同一个实例
}  // scope 结束，Scoped 对象被 Dispose
```

### 4.1 生命周期陷阱

```csharp
// 危险：Singleton 持有 Transient 依赖
// Singleton 的 Transient 成员永远不会被回收，变成事实上的 Singleton
public class SingletonService
{
    private readonly TransientService _transient;  // ← 生命周期被提升！
    public SingletonService(TransientService transient)
    {
        _transient = transient;  // 仅在构造时创建一次
    }
}
```

---

## 5. DI 与设计原则

### 5.1 SOLID 中的体现

| 原则 | DI 如何体现 |
|------|------------|
| **S**ingle Responsibility | 依赖的职责由各自的实现类承担 |
| **O**pen-Closed | 新增实现不修改消费代码 |
| **L**iskov Substitution | 注入点使用接口，任何实现都可替换 |
| **I**nterface Segregation | 注入点使用最小化接口 |
| **D**ependency Inversion | 高层模块依赖抽象（接口），不依赖具体类 |

### 5.2 DI 与设计模式的关系

```
依赖注入本质上是 Strategy 模式在依赖管理上的应用：
- Strategy：角色通过接口持有可替换的算法/策略
- DI：将策略的实例化和注入从角色中移出

DI 容器在内部使用 Factory 模式：
- 容器 = Abstract Factory
- 注册的类型 = 具体工厂的产品
```

### 5.3 OCP 的完整闭环

```csharp
// 1. 定义抽象（接口/抽象类）
public interface IPaymentProcessor { void Pay(decimal amount); }

// 2. 消费方依赖抽象
public class OrderService
{
    private readonly IPaymentProcessor _payment;

    public OrderService(IPaymentProcessor payment)  // 构造注入
    {
        _payment = payment;
    }

    public void Checkout(Order order)
    {
        _payment.Pay(order.Total);
    }
}

// 3. 新需求：支持比特币支付 → 只加一个新类，不改 OrderService
public class BitcoinProcessor : IPaymentProcessor
{
    public void Pay(decimal amount) { /* BTC 支付逻辑 */ }
}

// 4. 容器注册即可启用
container.Register<IPaymentProcessor, BitcoinProcessor>();
// OrderService 代码零改动
```

---

## 6. 实际应用场景

### 6.1 ASP.NET Core 内置 DI

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 注册服务
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.AddScoped<IUserRepository, EfUserRepository>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();

// 控制器自动获得注入
public class UsersController : ControllerBase
{
    private readonly IUserRepository _repo;
    private readonly IEmailSender _email;

    public UsersController(IUserRepository repo, IEmailSender email)
    {
        _repo = repo;
        _email = email;
    }
}
```

### 6.2 Unity 游戏中的 DI

```csharp
// 简单的手动 DI（无框架）
public class GameInstaller : MonoBehaviour
{
    void Awake()
    {
        // 组装依赖
        var input = new KeyboardInput();
        var physics = new PhysicsEngine();
        var renderer = new SpriteRenderer();

        var player = new PlayerController(input, physics);
        var enemy = new EnemyAI(physics, renderer);

        // 注入到全局服务定位器（适用于小型项目）
        ServiceLocator.Register<IPlayerController>(player);
        ServiceLocator.Register<IEnemyAI>(enemy);
    }
}

// 大型项目中可使用：
// - Zenject / Extenject（Unity 专用 DI 框架）
// - VContainer（轻量级，高性能）
// - 手动 Pure DI（无框架，最简单）
```

---

## 7. 常见面试题

### 7.1 概念题

1. **什么是依赖注入？**→ 将依赖的创建和管理从消费类中移出，由外部注入。是 IoC（控制反转）的一种实现形式。

2. **DI 解决了什么问题？**→ 解耦、可测试性、遵循 SOLID（特别是 DIP 和 OCP）。

3. **IoC 和 DI 的关系？**→ IoC 是原则（控制反转），DI 是实现 IoC 的一种方式。其他 IoC 实现包括：Service Locator、Factory Pattern、Template Method。

### 7.2 对比题

4. **构造注入 vs 属性注入 vs 方法注入？**
   - 构造注入：强制依赖，创建后不可变 → **首选**
   - 属性注入：可选依赖，可运行时修改 → 仅在确实可选时使用
   - 方法注入：仅在一次方法调用中需要 → 少见

5. **DI 容器 vs 手动注入（Pure DI）？**
   - 容器：自动解析依赖图，配置集中 → 适合中大型项目
   - Pure DI：手动在入口组装 → 编译时安全，无魔法 → 适合小型项目

### 7.3 陷阱题

6. **DI 容器是必须的吗？**→ 不是。DI 是一个模式/过程，容器只是工具。小型项目手动注入更简单直接。

7. **Service Locator 和 DI 的区别？**→ SL 是主动获取依赖（Pull），DI 是被动接收依赖（Push）。SL 隐藏依赖，违反显式原则，是反模式。

8. **Singleton 服务中能否注入 Scoped 服务？**→ 不能，Scoped 生命周期短于 Singleton，会导致 Scoped 实例被提前释放或提升为 Singleton。

9. **循环依赖如何处理？**→ 重构设计（引入中间抽象、拆分职责）。 `Lazy<T>` 是临时绕过，不推荐。
