---
title: "C# 集合框架深度解析"
tags:
  - csharp
  - collections
type: source
updated: 2026-06-02
source_files:
  - drafts/My_Vault/02_Knowledge/01_Language/常用接口/Tech_Csharp_常用接口_集合接口.md
  - drafts/My_Vault/files/C Sharp List优化 -- table.md
  - drafts/My_Vault/files/C sharp List扩容.md
  - drafts/My_Vault/files/C_Sharp_IList_IReadOnlyList.md
  - drafts/My_Vault/files/C sharp 重写list.md
---

# C# 集合框架深度解析

## 1. 集合接口层次结构

C# 集合接口形成清晰的继承层次，从基础迭代到完整可变列表：

```
IEnumerable (非泛型)
└── IEnumerable<T>       ← 可被 foreach 遍历
    └── IReadOnlyCollection<T>  ← 只读 + Count
        └── IReadOnlyList<T>    ← 只读 + 索引访问
    └── ICollection<T>    ← 可变 + Add/Remove
        ├── IList<T>      ← 可变 + 索引访问
        ├── ISet<T>       ← 集合（无重复）
        └── IDictionary<K,V>  ← 键值对
```

---

## 2. IEnumerable<T> —— 一切集合的根

所有可通过 `foreach` 遍历的类型都实现 `IEnumerable<T>`：

```csharp
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<T> : IEnumerator, IDisposable
{
    T Current { get; }
    bool MoveNext();
    void Reset();
}
```

### 2.1 yield return —— 迭代器语法糖

```csharp
// 编译器将 yield 方法转换为状态机，实现 IEnumerator<T>
public static IEnumerable<int> Fibonacci(int count)
{
    int a = 0, b = 1;
    for (int i = 0; i < count; i++)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

// 使用
foreach (var n in Fibonacci(10))
    Console.WriteLine(n);  // 0 1 1 2 3 5 8 13 21 34

// yield break —— 提前终止迭代
public static IEnumerable<int> RangeWithStop(int start, int count, int stopAt)
{
    for (int i = 0; i < count; i++)
    {
        if (start + i >= stopAt) yield break;
        yield return start + i;
    }
}
```

### 2.2 LINQ 延迟执行

```csharp
var query = numbers.Where(n => n > 5).Select(n => n * 2);
// 此时尚未执行任何计算 —— 只在枚举时才会执行

var result = query.ToList();  // 物化：此时才执行过滤和投影
```

---

## 3. ICollection<T> —— 可变集合

```csharp
public interface ICollection<T> : IEnumerable<T>
{
    int Count { get; }
    bool IsReadOnly { get; }
    void Add(T item);
    bool Remove(T item);
    void Clear();
    bool Contains(T item);
    void CopyTo(T[] array, int arrayIndex);
}
```

---

## 4. IList<T> —— 索引访问的可变列表

```csharp
public interface IList<T> : ICollection<T>, IEnumerable<T>
{
    T this[int index] { get; set; }  // 索引器
    int IndexOf(T item);
    void Insert(int index, T item);
    void RemoveAt(int index);
}
```

关键方法：

| 方法 | 说明 |
|------|------|
| `Add(T)` | 追加到末尾 |
| `Insert(int, T)` | 指定位置插入 |
| `Remove(T)` | 移除第一个匹配元素 |
| `RemoveAt(int)` | 按索引移除 |
| `IndexOf(T)` | 查找索引，未找到返回 -1 |
| `Clear()` | 清空 |
| `Contains(T)` | 判断是否存在 |

---

## 5. IReadOnlyList<T> —— 只读索引列表

```csharp
public interface IReadOnlyList<out T> : IReadOnlyCollection<T>, IEnumerable<T>
{
    T this[int index] { get; }
}
```

**设计意图**：表达"我只读，不会修改"的语义，让调用方无法意外修改数据。

```csharp
// 好的实践：接受 IReadOnlyList<T> 表达纯消费
int Sum(IReadOnlyList<int> numbers)
{
    int total = 0;
    for (int i = 0; i < numbers.Count; i++)
        total += numbers[i];
    return total;
}

// 调用
var list = new List<int> { 1, 2, 3 };
int result = Sum(list);  // List<T> 实现了 IReadOnlyList<T>
```

---

## 6. List<T> 内部实现与扩容

### 6.1 内部结构

`List<T>` 内部维护一个 `T[]` 数组和 `_size` 字段（实际元素数量）。`Capacity` 是数组的长度，`Count` 是实际元素数。

### 6.2 扩容策略

默认扩容策略是**翻倍**：当 `Count == Capacity` 时，新 Capacity = 旧 Capacity × 2（初始容量为 0 时，第一次 Add 后容量变为 4）。

```csharp
var list = new List<int>();
Console.WriteLine(list.Capacity);  // 0
list.Add(1);
Console.WriteLine(list.Capacity);  // 4
list.AddRange(new int[5]);
Console.WriteLine(list.Capacity);  // 8（翻倍）
```

### 6.3 扩容的代价

| 容量状态 | 问题 | 后果 |
|----------|------|------|
| **容量小** | 频繁扩容 | 旧数组被抛弃 → GC 压力；数组复制开销 |
| **容量大** | 单次扩容复制大量元素 | 内存峰值翻倍（旧数组 + 新数组同时存在） |

### 6.4 优化策略

```csharp
// 策略1：预分配容量 —— 知道大小时最推荐
var list = new List<int>(capacity: 1000);  // 避免 0→4→8→16… 的扩容链

// 策略2：手动设置 Capacity
list.Capacity = 1000;  // 一步到位

// 策略3：TrimExcess —— 释放多余空间
list.TrimExcess();  // Capacity = Count（仅在确实不需要增长时用）

// 策略4：对于超大列表，考虑自定义集合
// —— 分块存储（ArrayPool 租赁 + 多个子数组）或使用 Memory<T>/Span<T>
```

### 6.5 自定义 List 思路

```csharp
public class ChunkedList<T>
{
    private const int ChunkSize = 4096;
    private readonly List<T[]> _chunks = new();
    private int _count = 0;

    public void Add(T item)
    {
        if (_count == _chunks.Count * ChunkSize)
            _chunks.Add(new T[ChunkSize]);

        int chunkIndex = _count / ChunkSize;
        int offset = _count % ChunkSize;
        _chunks[chunkIndex][offset] = item;
        _count++;
    }

    public T this[int index]
    {
        get => _chunks[index / ChunkSize][index % ChunkSize];
        set => _chunks[index / ChunkSize][index % ChunkSize] = value;
    }

    public int Count => _count;
}
```

---

## 7. 其他集合类型速览

| 类型 | 特征 | 适用场景 |
|------|------|----------|
| `Dictionary<K,V>` | 哈希表，O(1) 查找 | 键值映射 |
| `HashSet<T>` | 无重复元素，O(1) 查找 | 去重、集合运算 |
| `SortedDictionary<K,V>` | 红黑树，按键排序 | 需要排序的键值映射 |
| `SortedSet<T>` | 红黑树，排序集合 | 需要排序的去重集合 |
| `Queue<T>` | 先进先出 (FIFO) | 任务队列、消息缓冲 |
| `Stack<T>` | 后进先出 (LIFO) | 撤销栈、DFS |
| `LinkedList<T>` | 双向链表 | 频繁中间插入/删除 |

### 7.1 线程安全集合

```csharp
// System.Collections.Concurrent
ConcurrentDictionary<K,V>  // 线程安全字典
ConcurrentQueue<T>         // 线程安全队列
ConcurrentStack<T>         // 线程安全栈
ConcurrentBag<T>           // 线程安全无序集合
BlockingCollection<T>      // 阻塞队列（生产者-消费者）
Channel<T>                 // 异步生产者-消费者（推荐）
```

---

## 8. List<T> 使用 struct vs class

当 List 存储大量元素时，类型选择至关重要：

```csharp
// 存储 class —— N 个堆对象 + N 个引用
class PointClass { public float X, Y; }
var classes = new List<PointClass>(1_000_000);
// GC Mark：需扫描 100 万个对象
// GC Compact：需移动 100 万个对象，并更新引用地址

// 存储 struct —— 1 个堆对象（内部数组）
struct PointStruct { public float X, Y; }
var structs = new List<PointStruct>(1_000_000);
// GC Mark：只需跟踪 1 个对象（内部数组）
// 数据连续，CPU 缓存友好
```

**选择指南**：

1. struct 适合 ≤16 bytes 的不可变小数据
2. struct 适合生命周期短、数量极大的情况（减少 GC 压力）
3. class 适合大小不确定、需要继承/多态的场景

## 8.1 Span<T> 与集合

C# 7.2+ 引入的 `Span<T>` 和 `ReadOnlySpan<T>` 允许以零分配方式操作连续内存：

```csharp
// 对 List 的切片操作 —— 零分配
List<int> list = [1, 2, 3, 4, 5, 6];
Span<int> slice = CollectionsMarshal.AsSpan(list)[2..4];  // 引用 [3, 4]，无复制

// 高效操作 List 内部数组
Span<int> span = CollectionsMarshal.AsSpan(list);
span[0] = 42;  // 直接修改 List 内部数组
```

## 8.2 ArrayPool<T> —— 减少数组分配

```csharp
// 租用临时数组 —— 用完归还，避免 GC
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    // 使用 buffer
    stream.Read(buffer, 0, buffer.Length);
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);  // 归还池中
    // 注意：归还后不要再使用 buffer！
}
```
4. 如果 struct 包含引用类型字段，优势减弱

---

## 9. foreach vs for 性能对比

```csharp
// foreach —— 每次都通过枚举器获取
foreach (var item in list) { Process(item); }

// for —— 直接索引访问，枚举器开销更小
for (int i = 0; i < list.Count; i++) { Process(list[i]); }
```

| 维度 | foreach | for |
|------|---------|-----|
| List<T> | 枚举器有轻微开销 | 索引直接，更快 |
| 数组 | 编译器优化，相等 | 编译器优化，相等 |
| Span<T> | 支持（foreach ref） | 支持 |
| IEnumerable | 必须用 | 不可用 |

---

## 10. 常用集合选择决策树

```
需要键值映射？
  ├── 是 → 需要排序？
  │        ├── 是 → SortedDictionary<K,V>
  │        └── 否 → Dictionary<K,V>
  └── 否 → 需要无重复？
           ├── 是 → 需要排序？
           │        ├── 是 → SortedSet<T>
           │        └── 否 → HashSet<T>
           └── 否 → 需要队列语义？
                    ├── 是 → FIFO？ → Queue<T>
                    │        LIFO？ → Stack<T>
                    └── 否 → 需要频繁中间插入？
                             ├── 是 → LinkedList<T>
                             └── 否 → List<T>（默认首选）
```

---

## 11. 面试要点

1. **IEnumerable → ICollection → IList 层次**：IEnumerable 只读遍历，ICollection 增加 Add/Remove/Count，IList 增加索引访问
2. **List 扩容机制**：翻倍扩容；问题是小容量频繁扩容（GC）和大容量复制开销
3. **struct 在集合中的优势**：减少堆分配，GC 友好，CPU 缓存友好
4. **foreach vs for**：List 上 for 略快（枚举器开销）；编译器对数组做了优化使二者相近
5. **IReadOnlyList 的意义**：语义上表达只读意图，防止调用方意外修改
6. **自定义集合**：当默认行为不满足时（如超大列表避免翻倍复制），可重写或使用 ArrayPool

## 12. IDisposable — 释放模式与集合

`IDisposable` 接口为资源管理提供统一的释放约定：

```csharp
public interface IDisposable
{
    void Dispose();
}
```

### 12.1 using 语法

```csharp
// using 声明（C# 8.0+）— 作用域结束时自动 Dispose
using var stream = new FileStream("data.bin", FileMode.Open);
// 方法结束时自动调用 stream.Dispose()

// using 语句块 — 块结束时 Dispose
using (var conn = new SqlConnection(connStr))
{
    conn.Open();
}  // conn.Dispose() 在此调用

// 多个资源 — 逆序释放（b → a）
using var a = new A();
using var b = new B();
```

### 12.2 Dispose(bool) 标准模式

```csharp
public class ResourceHolder : IDisposable
{
    private bool _disposed;
    private Stream? _managedResource;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // 释放托管资源
            _managedResource?.Dispose();
            _managedResource = null;
        }

        // 释放非托管资源（无论 disposing 是 true 还是 false）
        _disposed = true;
    }

    ~ResourceHolder()
    {
        Dispose(false);  // 析构函数仅释放非托管资源
    }
}
```

### 12.3 集合中的 IDisposable

- `IEnumerator<T>` 继承 `IDisposable` — foreach 自动释放枚举器
- 当集合持有 `IDisposable` 元素时，集合本身也应实现 `IDisposable` 进行级联释放
- `ConcurrentBag<T>` / `BlockingCollection<T>` 实现了 `IDisposable`

### 12.4 交叉引用

- [[csharp-memory-gc-摘要|C# 内存与GC]] — Dispose 模式与析构函数的配合
- [[csharp-serialization-摘要|C# 序列化与IO]] — Stream 与 IDisposable
