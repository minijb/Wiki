---
title: "C# 内存管理与GC深度解析"
tags:
  - csharp
  - gc
  - memory
  - struct
  - boxing
  - idisposable
type: language
created: 2026-06-02
source_files:
  - drafts/My_Vault/02_Knowledge/01_Language/Csharp/Csharp_Struct 如何减少装箱造成的GC.md
  - drafts/My_Vault/02_Knowledge/01_Language/注意/00_CSharp_匿名函数GC问题.md
  - drafts/My_Vault/files/C Sharp WeakReference.md
  - drafts/My_Vault/files/C Sharp IDispose.md
  - drafts/My_Vault/files/面试 - csharp.md
  - drafts/My_Vault/files/面试准备.md
---

# C# 内存管理与GC深度解析

## 1. 托管堆与栈

C# 内存分为两大区域：

| 区域 | 存储内容 | 管理方式 | 回收 |
|------|----------|----------|------|
| **Stack（栈）** | 值类型局部变量、引用（指针）、方法参数 | 方法调用帧自动压栈/弹栈 | 自动，方法返回即释放 |
| **Heap（堆）** | 引用类型对象（class、string、数组等）、装箱的值类型 | CLR 托管 | GC 定时回收 |

```csharp
void Example()
{
    int x = 42;           // 栈：值类型
    string s = "hello";   // 栈：引用 | 堆："hello" 对象
    MyClass obj = new();  // 栈：引用 | 堆：MyClass 实例
}  // 方法返回后，栈变量全部释放；堆对象等待 GC
```

---

## 2. GC（垃圾回收）机制

### 2.1 回收时机

- 堆内存分配时内存不足**自动触发**
- GC 会自动运行，不同平台频率不同
- 可手动强制：`GC.Collect()`（**不推荐**，仅在极少数场景使用）

### 2.2 回收的代价

1. **遍历扫描**：遍历堆上所有存活对象 → 对象越多越慢
2. **内存碎片化**：回收后产生碎片，影响后续分配
3. **压缩（Compact）**：移动存活对象以消除碎片 → 需要更新所有引用地址

### 2.3 分代回收

.NET GC 使用分代策略：

| 代 | 特点 | 回收频率 |
|-----|------|----------|
| **Gen 0** | 新分配的对象，短期存活 | 最频繁，速度快 |
| **Gen 1** | 从 Gen 0 幸存的对象 | 中等 |
| **Gen 2** | 长期存活对象（静态字段等） | 很少，代价最高 |

```csharp
// 查看当前 GC 代数
Console.WriteLine(GC.MaxGeneration);  // 通常是 2
Console.WriteLine(GC.GetGeneration(obj));  // obj 的代数
```

---

## 3. 装箱（Boxing）与拆箱（Unboxing）

### 3.1 装箱原理

装箱：将值类型实例转换为 `object` 或接口类型。

步骤：
1. 在托管堆分配内存（大小 = 值类型大小 + 对象头/方法表指针）
2. 将值类型数据复制到堆内存
3. 返回指向堆内存的引用

```csharp
int i = 42;
object o = i;  // 装箱：堆分配 + 复制
```

**性能代价**：
- 一次堆分配（触发 GC 压力）
- 一次内存复制
- 拆箱时的类型检查 + 再次复制

### 3.2 装箱发生的场景

```csharp
// 场景1：将 struct 当作 object 传递
void Print(object obj) => Console.WriteLine(obj);
Print(42);  // 装箱！

// 场景2：struct 调用未重载的基类方法
struct MyStruct { public int X; }
var s = new MyStruct { X = 5 };
s.ToString();   // 未重载？装箱！—— MyStruct 如果不重载 ToString，调用的是 ValueType.ToString()，需要装箱

// 场景3：struct 作为接口类型的参数传递（非泛型）
void Process(IComparable c) { /* ... */ }
Process(42);  // 装箱！int → IComparable

// 场景4：yield return 的值类型
IEnumerator GetEnumerator() { yield return 0; }  // 装箱！用 yield return null 替代
```

### 3.3 避免装箱的策略

**策略1：重载关键方法**

```csharp
struct MyStruct
{
    public int X;
    public override string ToString() => X.ToString();  // 避免装箱
    public override int GetHashCode() => X.GetHashCode();
    public override bool Equals(object obj) => obj is MyStruct other && X == other.X;
}
```

**策略2：泛型约束替代接口参数**

```csharp
// 装箱版本
void ProcessOld(IComparable a, IComparable b) { /* a.CompareTo(b) — 两次装箱 */ }

// 泛型版本 —— 零装箱
void ProcessNew<T>(T a, T b) where T : IComparable<T>
{
    a.CompareTo(b);  // 通过约束调用，不需要装箱
}
```

**策略3：统一接口提前拆箱**

```csharp
interface IData { int Value { get; } }
struct DataA : IData { public int Value => 1; }
struct DataB : IData { public int Value => 2; }

// 装箱发生在外层调用，内部不再重复装箱
void Process(IData data)
{
    int v = data.Value;  // 仅一次拆箱，后续操作无装箱
}
```

---

## 4. Struct vs Class 内存对比

| 维度 | struct（值类型） | class（引用类型） |
|------|-----------------|-------------------|
| 存储位置 | 栈或内联在容器中 | 堆 |
| GC 跟踪 | 不需要 | 需要扫描 |
| 复制方式 | 位拷贝（memcpy） | 引用复制 |
| 继承 | 不支持 | 支持 |
| 默认值 | 所有字段零初始化 | null |
| 适合场景 | 小型不可变数据（< 16 bytes） | 复杂可变对象 |

### 4.1 List 中使用 struct

```csharp
// 在 List 中存储 class：每次 GC 需扫描一百万个对象
List<PointClass> classList = new(1_000_000);
// → 100万个独立堆对象 + 100万个引用需要 GC 跟踪

// 在 List 中存储 struct：仅 List 内部数组是堆对象
List<PointStruct> structList = new(1_000_000);
// → 仅1个托管堆对象（List 内部的 T[] 数组）
// GC 只需跟踪这一个对象
// 数据连续存储，CPU 缓存友好
```

**警示**：struct 过大会导致复制开销过高（每次赋值/传参都是完整拷贝），一般建议 ≤ 16 bytes。

---

## 5. 匿名函数（Lambda）与闭包GC

### 5.1 闭包原理

Lambda 表达式捕获外部变量时，编译器生成一个**匿名类**，将捕获的变量作为该类的字段存储在堆上：

```csharp
// 原始代码
public void Process(string name)
{
    int count = 0;
    Action action = () => Console.WriteLine($"{name}: {count++}");
    action();
}

// 编译器生成（简化）
private sealed class <>c__DisplayClass0_0
{
    public string name;
    public int count;
    internal void <Process>b__0()
    {
        Console.WriteLine($"{name}: {count++}");
    }
}

public void Process(string name)
{
    var closure = new <>c__DisplayClass0_0();  // 堆分配！
    closure.name = name;
    closure.count = 0;
    closure.<Process>b__0();
}
```

### 5.2 GC 问题场景

```csharp
// 问题：每次调用都创建新的闭包对象
public void LoadImage(string path)
{
    // 闭包捕获了 charBustComponent 和 cachedTextures
    ImageManager.LoadImage(path, (resPath, resource) =>
    {
        // 此闭包对象在堆上分配
        charBustComponent.SwitchPart(resource.tex, partName);
        cachedTextures.Add(resPath, resource.tex);
    });
}
```

### 5.3 优化策略

- 高频调用的 Lambda → 改用实例方法或缓存委托
- 避免在 Update/FixedUpdate 中创建闭包
- 用对象池或预分配回调替代动态 Lambda

---

## 6. WeakReference 弱引用

```csharp
// 弱引用：允许 GC 回收目标对象，不影响对象生命周期
var strongRef = new LargeObject();          // 强引用 —— GC 不会回收
var weakRef = new WeakReference<LargeObject>(strongRef);

strongRef = null;  // 移除强引用

if (weakRef.TryGetTarget(out var obj))
{
    // 对象仍存活 —— GC 尚未回收
    Use(obj);
}
else
{
    // 对象已被 GC 回收
}

// 常见用途：缓存、事件订阅（避免内存泄漏）
```

### 6.1 WeakReference 缓存模式

```csharp
public class WeakCache<TKey, TValue> where TValue : class
{
    private readonly Dictionary<TKey, WeakReference<TValue>> _cache = new();

    public void Add(TKey key, TValue value)
    {
        _cache[key] = new WeakReference<TValue>(value);
    }

    public bool TryGet(TKey key, out TValue? value)
    {
        if (_cache.TryGetValue(key, out var wr) && wr.TryGetTarget(out value))
            return true;

        _cache.Remove(key);  // 清理已死亡的弱引用
        value = null;
        return false;
    }

    // 清理所有已死亡的弱引用
    public void Cleanup()
    {
        foreach (var key in _cache.Where(kvp => !kvp.Value.TryGetTarget(out _)).Select(kvp => kvp.Key).ToList())
            _cache.Remove(key);
    }
}
```

---

## 7. IDisposable 与资源管理

### 7.1 托管资源 vs 非托管资源

| 类型 | 说明 | 示例 |
|------|------|------|
| **托管资源** | CLR 管理的对象 | `new` 出来的任何类实例 |
| **非托管资源** | CLR 不管理的，跨平台调用获得 | 文件句柄、Socket、数据库连接、GDI+ 对象、Win32 API |

### 7.2 Dispose 模式

```csharp
public class ResourceHolder : IDisposable
{
    private bool _disposed = false;
    private Stream? _managedStream;  // 托管资源（也实现了 IDisposable）
    private IntPtr _nativeHandle;    // 非托管资源

    public ResourceHolder()
    {
        _managedStream = new FileStream("data.bin", FileMode.Open);
        _nativeHandle = NativeMethods.CreateHandle();
    }

    // 公共 Dispose —— 使用者调用
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);  // 告诉 GC 不需要调用析构函数
    }

    // 受保护的 Dispose(bool) —— 子类可重写
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // 释放托管资源（这些资源可能也实现了 IDisposable）
            _managedStream?.Dispose();
            _managedStream = null;
        }

        // 释放非托管资源（无论 disposing 是 true 还是 false 都要释放）
        if (_nativeHandle != IntPtr.Zero)
        {
            NativeMethods.DestroyHandle(_nativeHandle);
            _nativeHandle = IntPtr.Zero;
        }

        _disposed = true;
    }

    // 析构函数 —— GC 回收前的最后保障
    ~ResourceHolder()
    {
        Dispose(disposing: false);
        // 此时托管资源可能已无法访问，只释放非托管资源
    }
}
```

### 7.3 using 语法

```csharp
// using 声明（C# 8.0+）—— 作用域结束时自动 Dispose
using var stream = new FileStream("data.bin", FileMode.Open);
// ... 使用 stream
// 方法结束时自动调用 stream.Dispose()

// using 语句块 —— 块结束时 Dispose
using (var conn = new SqlConnection(connStr))
{
    conn.Open();
    // ...
}  // conn.Dispose() 在此调用

// 多个资源
using var a = new A();
using var b = new B();
// 逆序释放：b → a
```

### 7.4 级联 Dispose

当一个类持有实现了 `IDisposable` 的成员时，该类也应实现 `IDisposable`，在其 `Dispose` 中调用成员 Dispose。

---

## 8. GC 优化实践总结

| 策略 | 具体措施 |
|------|----------|
| **减少堆分配** | 用 StringBuilder 替代大量字符串拼接；struct 替代小型数据的 class |
| **减少装箱** | 重载 ToString/GetHashCode/Equals；用泛型替代 object 参数 |
| **预分配容量** | `new List<T>(capacity)`、`new StringBuilder(capacity)`，避免扩容导致的旧数组被抛弃 |
| **对象池** | 频繁创建/销毁的对象（如子弹、粒子）用池管理 |
| **避免闭包GC** | 高频路径避免 Lambda 捕获变量；缓存委托 |
| **及时释放** | 非托管资源实现 IDisposable + using |
| **弱引用缓存** | 缓存大数据对象时用 WeakReference，允许 GC 在内存紧张时回收 |
| **时机控制** | 避免在关键帧执行 GC；在加载/转场时主动 GC（需谨慎） |

---

## 9. 面试要点

1. **GC 原理**：分代回收，Gen 0/1/2，标记-压缩-更新引用
2. **装箱**：值类型 → object/interface，堆分配 + 复制；避免方法：重载、泛型、统一接口
3. **struct vs class**：值类型无 GC 压力但复制有成本；引用类型灵活但 GC 需要跟踪每个实例
4. **闭包GC**：Lambda 捕获外部变量 → 编译器生成匿名类（堆分配）→ 每次调用可能产生新对象
5. **IDisposable vs Finalizer**：Dispose 用于及时释放；析构函数是 GC 回收的最后保障（只处理非托管资源）；`GC.SuppressFinalize` 防止重复清理
6. **WeakReference**：允许 GC 回收，用于缓存等非强制性引用场景
