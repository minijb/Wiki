---
title: "C++ 核心语法"
type: source
updated: 2026-06-02
tags: [cpp, stl, move-semantics, memory, smart-pointers, hash, value-categories]
aliases: [C++语法, C++核心, C++ STL]
description: C++ 核心语法速查：移动语义与右值引用、完美转发（std::forward）、STL 容器与适配器全景、哈希自定义（std::hash 特化/函数对象）、智能指针（unique/shared/weak）、内存分区与分配器、虚表与对象布局、类型转换、编译过程
---

# C++ 核心语法

## 移动语义与右值引用

### 左值与右值

- **左值**：有地址、可持久存在的对象（如变量 `int x = 5;` 中的 `x`）
- **右值**：没有地址的临时对象，要么是字面常量，要么是表达式求值过程中创建的临时对象

右值的两个关键特征：
1. 该对象将要被销毁
2. 该对象没有其他用户再使用它

### 移动的本质

C++ 中 `=` 的默认操作是拷贝。移动操作相当于数据的**转移**——原位置不再持有该值，避免了不必要的深拷贝。

```cpp
int val{ 0 };
int&& rRef0{ getTempValue() };  // OK，引用临时对象
int&& rRef1{ val };             // Error，不能引用左值
int&& rRef2{ std::move(val) };  // OK，std::move 将左值标记为右值
```

编译器匹配右值引用的情况：
1. 语句执行完毕后就会被自动销毁的临时对象
2. 由 `std::move` 标记的非 const 对象

### 实现移动构造与移动赋值

```cpp
class MyClass {
public:
    MyClass()
        : val{ 998 }
    {
        name = new char[] { "Peter" };
    }

    // 移动构造函数
    MyClass(MyClass&& other) noexcept
        : val{ other.val }        // 转移简单值
        , name{ other.name }      // 转移指针所有权
    {
        other.val = 0;
        other.name = nullptr;     // 清空源对象，防止析构时双重释放
    }

    // 移动赋值运算符
    MyClass& operator=(MyClass&& other) noexcept {
        if (this != &other) {
            delete[] name;            // 先释放自身原有资源
            val = other.val;
            name = other.name;
            other.val = 0;
            other.name = nullptr;
        }
        return *this;
    }

    ~MyClass() {
        delete[] name;
    }

private:
    int val;
    char* name;
};

MyClass A{};
MyClass B{ std::move(A) };  // 调用移动构造函数
```

`noexcept` 在移动构造/赋值中是**必须的**：标准库容器（如 `std::vector` 扩容时）通过 `std::move_if_noexcept` 检测，若移动操作未标记 `noexcept` 则退化为拷贝，失去性能优势。

### 完美转发

完美转发在函数模板中保持参数的原始类型和引用属性（左值/右值、const/非const）原样传递给另一个函数。

- **转发引用（万能引用）**：模板中的 `T&&` 既可以绑定左值也可以绑定右值
- `std::forward<T>`：保持参数的值类别进行转发

```cpp
void process(int& x) {
    std::cout << "Lvalue process: " << x << std::endl;
}

void process(int&& x) {
    std::cout << "Rvalue process: " << x << std::endl;
}

template <typename T>
void forwardToProcess(T&& arg) {
    process(std::forward<T>(arg));  // 保持 arg 的值类别
}

int main() {
    int a = 42;
    forwardToProcess(a);            // 转发左值 → process(int&)
    forwardToProcess(42);           // 转发右值 → process(int&&)
    forwardToProcess(std::move(a)); // 转发右值 → process(int&&)
}
```

## STL 容器全景

STL 容器分为三大类，底层实现决定了性能特征：

### 序列容器

| 容器 | 底层结构 | 特点 | 随机访问 | 头插入 | 尾插入 |
|------|---------|------|---------|--------|--------|
| `std::vector` | 动态数组 | 自动扩容（1.5x 或 2x），内存连续 | O(1) | O(n) | O(1) 平摊 |
| `std::array` | 固定数组 | 大小编译期确定，无堆分配 | O(1) | — | — |
| `std::deque` | 分段连续数组 | 双端 O(1) 插入，随机访问 | O(1) | O(1) | O(1) |
| `std::list` | 双向链表 | 任意位置 O(1) 插入删除 | — | O(1) | O(1) |
| `std::forward_list` | 单向链表 | 只有 push_front，节省内存 | — | O(1) | — |

**vector 扩容机制**：当 `size() == capacity()` 时，分配新内存（通常 1.5x 或 2x 旧容量），将旧元素**移动**（若移动构造为 `noexcept`）或拷贝到新地址，释放旧内存。这导致迭代器失效。

**deque 内部结构**：由一段段定量连续空间（缓冲区）组成，通过一块 map（指针数组）维护整体连续的假象。比 vector 的优势在于无需整体重新分配，代价是迭代器结构复杂。除非必要，优先使用 vector。

**list 特殊能力**：`std::list` 提供自己的 `sort()` 成员函数（归并排序），比通用 `std::sort()` 更高效，因为链表无法随机访问。

### 容器适配器

适配器在底层容器基础上提供受限接口：

| 适配器 | 默认底层容器 | 说明 |
|--------|-------------|------|
| `std::stack` | `deque` | LIFO，只暴露 push/pop/top |
| `std::queue` | `deque` | FIFO，push/pop/front/back |
| `std::priority_queue` | `vector` | 默认大根堆（`std::less`），可自定义比较器 |

```cpp
// priority_queue 模板签名
template<
    class T,
    class Container = std::vector<T>,
    class Compare = std::less<typename Container::value_type>
> class priority_queue;

// 小根堆
std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;

// 自定义比较器
auto cmp = [](int left, int right) { return (left ^ 1) < (right ^ 1); };
std::priority_queue<int, std::vector<int>, decltype(cmp)> pq(cmp);
```

### 关联容器（有序）

基于红黑树，key 不可修改。保持有序，操作 O(log n)。

| 容器 | key 重复 | 映射 |
|------|---------|------|
| `std::set` | 否 | 纯 key |
| `std::multiset` | 是 | 纯 key |
| `std::map` | 否 | key→value |
| `std::multimap` | 是 | key→value |

### 无序关联容器（哈希表）

基于哈希表，O(1) 平均查询。优先使用除非需要有序。

| 容器 | key 重复 | 映射 |
|------|---------|------|
| `std::unordered_set` | 否 | 纯 key |
| `std::unordered_multiset` | 是 | 纯 key |
| `std::unordered_map` | 否 | key→value |
| `std::unordered_multimap` | 是 | key→value |

**选择指南**：需要集合优先 `unordered_set`（最优增删查），需要有序用 `set`，有序且可重复用 `multiset`。

## 哈希自定义

`std::unordered_map` 和 `std::unordered_set` 需要两样东西：哈希函数 + 相等比较。

### 特化 std::hash

```cpp
struct Point {
    int x, y;
    bool operator==(const Point& other) const {
        return x == other.x && y == other.y;
    }
};

// 特化 std::hash
namespace std {
    template<>
    struct hash<Point> {
        size_t operator()(const Point& p) const noexcept {
            // 组合哈希：避免 (x,y) 和 (y,x) 冲突
            size_t h1 = hash<int>{}(p.x);
            size_t h2 = hash<int>{}(p.y);
            return h1 ^ (h2 << 1);
        }
    };
}

// 使用
std::unordered_set<Point> points;
```

### 传入自定义函数对象

```cpp
struct PointHash {
    size_t operator()(const Point& p) const noexcept {
        return std::hash<int>{}(p.x) ^ (std::hash<int>{}(p.y) << 1);
    }
};

struct PointEqual {
    bool operator()(const Point& a, const Point& b) const {
        return a.x == b.x && a.y == b.y;
    }
};

std::unordered_map<Point, std::string, PointHash, PointEqual> map;
```

### 哈希表扩容

负载因子（load factor）= `size() / bucket_count()`。当负载因子超过 `max_load_factor()`（默认一般为 1.0），触发扩容：
1. 创建新数组（通常 2x）
2. 重新哈希所有元素放入新位置
3. 所有迭代器失效

## 智能指针

| 类型 | 说明 |
|------|------|
| `std::unique_ptr` | 独占所有权，不可拷贝，可移动。析构自动 delete |
| `std::shared_ptr` | 共享所有权，引用计数。计数归零时 delete |
| `std::weak_ptr` | 不增加引用计数，用于打破循环引用。需 `lock()` 才能访问 |

```cpp
// unique_ptr
auto p = std::make_unique<MyClass>(args...);  // C++14+

// shared_ptr
auto sp = std::make_shared<MyClass>(args...);

// weak_ptr 打破循环引用
class B;
class A {
    std::shared_ptr<B> b_ptr;
};
class B {
    std::weak_ptr<A> a_ptr;  // 不增加引用计数，防止循环
};
```

## 内存管理

### 内存分区

| 区域 | 内容 |
|------|------|
| 代码区 | 可执行指令 |
| 数据区 (data) | 已初始化的全局/静态变量 |
| BSS 区 | 未初始化的全局/静态变量 |
| 堆 (heap) | 动态分配（malloc/new） |
| 栈 (stack) | 局部变量、函数参数 |
| 文件映射区 | 动态链接库、mmap 映射 |
| 内核区 | 操作系统保留 |

### malloc 底层原理

- `malloc` 分配的是**虚拟内存**，并非直接分配物理内存
- 分配后若未访问，虚拟内存不会映射到物理内存（延迟分配）
- 首次访问时触发**缺页中断**，OS 建立虚拟→物理映射
- 小内存（通常 < 128KB）通过 `brk()` 移动堆顶指针
- 大内存通过 `mmap()` 在文件映射区分配
- `free()` 通过 `brk()` 分配的内存不立即归还 OS，缓存于 malloc 内存池
- `free()` 通过 `mmap()` 分配的内存立即归还 OS

### new 的多种形式

```cpp
// 1. operator new — 分配空间 + 调用构造函数
auto* p1 = new MyClass();

// 2. placement new — 在已分配空间上构造对象（不分配内存）
alignas(MyClass) char buf[sizeof(MyClass)];
auto* p2 = new (buf) MyClass();

// 3. nothrow new — 不抛异常
auto* p3 = new (std::nothrow) MyClass();

// 4. new[] — 数组分配
auto* p4 = new MyClass[10];
```

### STL 分配器

`std::allocator` 两阶段操作：
1. `allocate(n)` — 申请原始内存空间
2. `construct(p, args...)` — 在已分配空间构造对象

释放：先 `destroy(p)` 再 `deallocate(p, n)`

两级分配器（SGI STL 经典实现）：
- 大于 128B：一级分配器，直接 `malloc`/`free`
- 小于等于 128B：二级分配器，内存池 + 自由链表

### 内存池

基本算法：
1. 申请一块内存区，按对象大小切分为多个 block
2. 维护空闲 block 链表，头指针标记第一个空闲块
3. 分配时取链表头 block，更新头指针
4. 释放时将 block 插回链表头部
5. 内存区满时开辟新区，用链表连接多块内存区

## 强制类型转换

| 转换 | 用途 | 安全性 |
|------|------|--------|
| `static_cast` | 隐式转换（数值、void*→T*） | 编译期检查 |
| `const_cast` | 移除/添加 const | 危险 |
| `dynamic_cast` | 向下转型（含虚函数的类） | 运行时检查，失败返回 nullptr（指针）或抛 std::bad_cast（引用） |
| `reinterpret_cast` | 任意指针互转 | 极危险，不保证可移植 |

## constexpr 与 const

```cpp
const double pi = 3.14159;        // 常量，初始化后不可变
constexpr double tau = 6.28318;   // 常量表达式，编译期确定值
constexpr double area = pi + 1;   // pi 不是 constexpr，此表达式不是编译期常量

constexpr int square(int x) {     // constexpr 函数：编译期求值（参数为编译期已知时）
    return x * x;
}
constexpr int val = square(10);   // 编译期求值
```

## 异常处理

```cpp
class BadArea {};  // 自定义异常类型

int area(int length, int width) {
    if (length <= 0 || width <= 0)
        throw BadArea{};
    return length * width;
}

try {
    int a = area(11, -5);
} catch (const BadArea&) {
    std::cout << "Bad argument of area()!\n";
} catch (const std::runtime_error& e) {
    std::cout << e.what() << '\n';
} catch (...) {  // 捕获所有异常
    std::cout << "Unknown error\n";
}
```

标准异常：`std::out_of_range`、`std::runtime_error("message")`。

## 编译过程

```
源代码(.cpp/.h) → 预处理 → 编译 → 汇编 → 链接 → 可执行文件
```

| 阶段 | 输出 | 工作 |
|------|------|------|
| 预处理 | `.i` | 宏展开、`#include`、条件编译 |
| 编译 | `.s` | C++→汇编，语法检查、模板实例化 |
| 汇编 | `.o`/`.obj` | 汇编→机器码 |
| 链接 | 可执行文件 | 符号解析、地址重定位 |

## 虚表与对象模型

### 普通对象内存布局

C++ 对象中包含的内容：
- 非静态成员变量
- 虚函数表指针 (vptr)

**不包含**在对象中的内容：
- 静态成员变量（类级别存储）
- 非虚成员函数（代码段，所有实例共享）
- 构造函数/析构函数（代码段）

```cpp
class Base {
    double _x;        // 8 字节
    virtual ~Base() {}
};
// sizeof(Base) = 16: _x(8) + vptr(8 in Linux, 8 in 64-bit VS) = 16
```

空类大小为 1 字节（C++ 要求不同对象有不同地址）。

### 单继承布局

```
Base:
  vptr → vtable (Base::vfunc1, Base::vfunc2)
  _base_data

Derived:
  vptr → vtable (Derived::vfunc1, Base::vfunc2)  // 重写的指向自己
  _base_data
  _derived_data
```

### 多重继承

```
Derived:
  vptr1 → vtable (B::vfunc 重写 + Derived::vfunc)
  _b_data
  vptr2 → vtable (C::vfunc 重写)
  _c_data
  _derived_data
```

多继承中有多张虚表，强制转换时 `this` 指针可能需要调整偏移。

### 虚继承与虚基类表

虚继承引入 `vbptr`（虚基类表指针），指向偏移值表，用于在运行时计算虚基类的正确位置。

**虚函数表**：编译期生成，存储在**只读数据段**。

**虚函数指针**：构造函数执行时动态绑定到对象的 vptr。

**为什么构造函数不能是虚函数**：调用虚函数需要 vptr，而在构造函数执行期间 vptr 还未初始化完成。

## 编译器优化与陷阱

### 初始化避免窄化

```cpp
double x {2.4};     // OK
int y {2.7};        // Error: 窄化转换
int z = 2.7;        // OK: 隐式截断为 2（不推荐）
```

### 常见整数除法陷阱

```cpp
double a = 9 / 5;    // 结果：1（整数除法）
double b = 9.0 / 5;  // 结果：1.8
```

### std::underlying_type

```cpp
enum class Color : uint8_t { Red, Green, Blue };
using Underlying = std::underlying_type_t<Color>;  // uint8_t
```

### std::tuple 结构化绑定

```cpp
auto t = std::make_tuple(1, "hello", 3.14);
int x = std::get<0>(t);
auto [a, b, c] = t;   // C++17 结构化绑定
```

## STL 常用 API 速查

### 算法使用技巧

```cpp
// 二分查找
bool found = std::binary_search(v.begin(), v.end(), target);
auto it = std::lower_bound(v.begin(), v.end(), target);  // 第一个 >= target
auto it2 = std::upper_bound(v.begin(), v.end(), target); // 第一个 > target

// 排序
std::sort(v.begin(), v.end());
std::sort(v.begin(), v.end(), std::greater<>());

// 三数之和问题剪枝
// 跳过重复结果：nums[i] == nums[i-1] → continue
// 跳过已处理状态：nums[i-1] == nums[i] → continue（此状态已出现过）
```

## 参见

- [[concepts/CSharp并发模型|C# 并发模型]] — 与 C++ 异步模型的对比
- [[concepts/CSharp值类型性能|C# 值类型性能]] — struct 内存布局对比
