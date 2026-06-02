---
title: "Lua 核心特性"
source_type: language-reference
tags: [lua, table, closure, metatable, coroutine, upvalue, gc]
created: 2026-06-02
updated: 2026-06-02
---

# Lua 核心特性

本文覆盖 Lua 语言的核心机制：Table 底层结构、闭包与 UpValue、元表系统、协程、值传递语义与循环控制。

---

## 1. Table 底层原理

### 1.1 双段结构

Lua 的 table 由**数组段**（array part）和**哈希段**（hash part）两部分组成：

- **数组段**：存储从 1 开始的连续整数键，以数组形式高效访问
- **哈希段**：存储所有非连续整数键、字符串键及其他类型键

```lua
local t = {[1]=1, 2, [3]=3, 4, [5]=5, [6]=6}
-- 数组段: {2, 4}       （实际索引 1→2, 2→4）
-- 哈希段: {[1]=1, [3]=3, [5]=5, [6]=6}
```

**分配原则**（来自 `The Implementation of Lua 5.0`）：
1. 数组空间至少有一半以上被利用（保证空间利用率）
2. 数组的后半部分内至少有一个元素（避免浪费空间）

因此 Lua 会优先将连续整数键放入数组段，剩下的放入哈希段。

### 1.2 Rehash 机制

Table 的内存大小动态分配，空间不够时扩展，利用率过低时收缩。

**Rehash 触发条件**：
- 插入新键时数组段或哈希段空间不足
- 大量删除后利用率低于阈值

**Rehash 过程**：

1. 计算数组部分已使用的键数量（`numusearray`）
2. 计算哈希部分中整数键的数量（`numusehash`）
3. 计算新键加入后数组部分应有的键数量（`countint`）
4. 通过 `computesizes` 计算最优数组段大小
5. 调用 `resize` 重新分配空间

```c
static void rehash(lua_State *L, Table *t, const TValue *ek) {
    int nums[MAXBITS+1];
    int nasize = numusearray(t, nums);    // 数组段已用
    int totaluse = nasize;
    totaluse += numusehash(t, nums, &nasize); // 哈希段整数键
    nasize += countint(ek, nums);             // 新键
    totaluse++;
    int na = computesizes(nums, &nasize);     // 最优数组大小
    resize(L, t, nasize, totaluse - na);      // 重新分配
}
```

### 1.3 `#` 运算符的陷阱

`#` 运算符对含有 nil 的 table 行为不可靠——它只保证在**没有空洞**的序列上返回正确长度。

```lua
local a = {1, 2, 3, nil, 5}
print(#a)  -- 可能是 3 或 5，取决于内部布局

-- 安全的长度获取
function table_len(t)
    local len = 0
    for _ in pairs(t) do
        len = len + 1
    end
    return len
end
```

**建议**：
- 数组尽量不包含 nil，删除元素用 `table.remove` 而非赋 nil
- 提前分配大小：`local tb = {nil, nil, nil}` 可预分配数组段空间
- 删除元素的 nil 赋值不会触发 rehash，但会影响 `#` 的结果

### 1.4 pairs vs ipairs vs 数值 for

| 遍历方式 | 适用范围 | 顺序保证 | 性能 | nil 行为 |
|----------|----------|----------|------|----------|
| `for i=1,#t` | 连续整数索引 | 确定，可控步长 | 最高 | 遇到 nil 中断 |
| `ipairs(t)` | 从 1 开始的连续整数索引 | 升序（1,2,3...） | 中等 | 遇到第一个 nil 停止 |
| `pairs(t)` | 所有键值对 | 不确定 | 较慢 | 可遍历含 nil 的键 |
| `next(t, k)` | 手动控制 | 依赖遍历过程 | — | — |

```lua
-- 数值 for：性能最高，可控方向和步长
for i = #array, 1, -1 do
    print(array[i])
end

-- ipairs：语法简洁，但遇 nil 即停
for i, v in ipairs(t) do
    print(i, v)
end

-- pairs：最通用，遍历所有键
for k, v in pairs(t) do
    print(k, v)
end
```

`pairs` 的输出顺序：先按索引输出数组段元素，再输出哈希段元素，哈希段元素按内部哈希算法排序（不保证连续）。

---

## 2. 闭包与 UpValue

### 2.1 UpValue 概念

Lua 将闭包捕获的外部局部变量称为 **UpValue**。Lua 维护一个全局栈，所有 UpValue 指向栈中的值。当变量离开作用域时，若仍有闭包引用，该值会被提升到堆上保存（closed 状态），否则随栈释放。

### 2.2 Open 与 Closed 状态

- **Open**：闭包创建时开启，UpValue 指向栈中活跃的变量
- **Closed**：当没有任何闭包引用该 UpValue 时关闭，资源被回收

多个闭包引用同一个外部局部变量时，它们**共享同一个 UpValue**：

```lua
function counter()
    local n = 0
    return function() n = n + 1; return n end,
           function() n = n - 1; return n end
end

inc, dec = counter()
print(inc())  -- 1
print(inc())  -- 2
print(dec())  -- 1  -- inc 和 dec 共享同一个 n
```

### 2.3 闭包运行机制

闭包执行时，通过 UpValue 指针链表找到所需的外部变量。UpValue 通过链表连接，闭包沿着链表遍历定位变量。

```lua
function F(x)
    -- x 作为 UpValue 被内层函数捕获，生命周期延长
    return function(y)
        return x + y  -- x 是 UpValue
    end
end

f = F(10)
print(f(5))  -- 15，x=10 被闭包持有
```

### 2.4 闭包的实用场景

- **迭代器**：创建带状态的遍历函数
- **回调封装**：为回调注入上下文数据
- **模块封装**：隐藏实现细节，只暴露接口

```lua
-- 带状态的迭代器
function range(from, to)
    return function()
        if from <= to then
            local v = from
            from = from + 1
            return v
        end
    end
end

for v in range(1, 5) do
    print(v)  -- 1, 2, 3, 4, 5
end
```

---

## 3. 元表系统 (Metatable)

### 3.1 概念

元表允许改变 table 的行为。当对 table 执行特定操作时，Lua 会查找其元表中对应的**元方法**来执行。

### 3.2 元方法一览

| 元方法 | 触发操作 | 签名 |
|--------|----------|------|
| `__index` | 读取不存在的键 | `function(table, key)` 或 table |
| `__newindex` | 赋值不存在的键 | `function(table, key, value)` 或 table |
| `__add` | `+` | `function(a, b)` |
| `__sub` | `-` | `function(a, b)` |
| `__mul` | `*` | `function(a, b)` |
| `__div` | `/` | `function(a, b)` |
| `__mod` | `%` | `function(a, b)` |
| `__pow` | `^` | `function(a, b)` |
| `__unm` | 取反 `-a` | `function(a)` |
| `__concat` | `..` | `function(a, b)` |
| `__len` | `#` | `function(a)` |
| `__eq` | `==` | `function(a, b)` |
| `__lt` | `<` | `function(a, b)` |
| `__le` | `<=` | `function(a, b)` |
| `__call` | 作为函数调用 | `function(table, ...)` |
| `__tostring` | `tostring()` | `function(table)` |
| `__metatable` | `getmetatable()` | 保护元表不被获取 |

### 3.3 `__index` 与 `__newindex` 详解

**`__index`**：当在表中找不到 key 时触发。可以是函数或表：
- 若为函数：`__index(table, key)` 被调用，返回结果作为查找值
- 若为表：在该表中继续查找 key

**`__newindex`**：当对表中不存在的 key 赋值时触发。可以是函数或表：
- 若为函数：`__newindex(table, key, value)` 被调用
- 若为表：值被写入该表而非原表

**绕过元方法**：
- `rawget(table, key)` — 直接读取，忽略 `__index`
- `rawset(table, key, value)` — 直接写入，忽略 `__newindex`

```lua
-- __index 实现原型继承
father = { prop1 = 2 }
father.__index = father  -- 关键：指向自身

son = { prop2 = 1 }
setmetatable(son, father)

print(son.prop1)  -- 2: 从 father 中查找
-- 流程：son 中找不到 prop1 → 调用 father.__index(son, "prop1")
--       由于 __index 是表，在 father 中找 "prop1" → 返回 2
```

### 3.4 运算符重载

```lua
local Vector = {}
Vector.__index = Vector

function Vector:new(x, y)
    return setmetatable({ x = x, y = y }, self)
end

function Vector.__add(a, b)
    return Vector:new(a.x + b.x, a.y + b.y)
end

function Vector.__tostring(v)
    return string.format("Vector(%d, %d)", v.x, v.y)
end

local v1 = Vector:new(1, 2)
local v2 = Vector:new(3, 4)
local v3 = v1 + v2
print(v3)  -- Vector(4, 6)
```

### 3.5 基于元表的 OOP

```lua
-- 基类定义
Object = {}
Object.__index = Object  -- 使实例能访问类方法

function Object:new()
    local obj = {}
    setmetatable(obj, self)  -- self = Object
    return obj
end

function Object:test()
    print("Object:test")
end

-- 继承
function Object:subClass(className)
    _G[className] = {}
    local cls = _G[className]
    setmetatable(cls, self)
    self.__index = self
    cls.base = self  -- 保留父类引用，用于多态
end

-- 使用
Object:subClass("GameObject")

function GameObject:Move()
    self.x = (self.x or 0) + 1
end

Object:subClass("Player")
-- Player 继承自 GameObject，GameObject 继承自 Object

-- 多态：调用父类方法
function Player:Move()
    self.base.Move(self)  -- 显式传入 self
end
```

**冒号语法糖**：`obj:method(args)` 等价于 `obj.method(obj, args)`，自动将调用者作为第一个参数（self）传入。

---

## 4. 协程 (Coroutine)

### 4.1 创建与运行

Lua 协程是非抢占式的协作式多任务，由程序主动让出执行权。

```lua
-- 方式一：coroutine.create + coroutine.resume
local co1 = coroutine.create(function()
    for i = 1, 3 do
        coroutine.yield(i)
    end
end)

local ok, val = coroutine.resume(co1)  -- ok=true, val=1
ok, val = coroutine.resume(co1)         -- ok=true, val=2
ok, val = coroutine.resume(co1)         -- ok=true, val=3
ok, val = coroutine.resume(co1)         -- ok=true, val=nil（协程结束）

-- 方式二：coroutine.wrap（返回函数，不返回 ok）
local co2 = coroutine.wrap(function()
    for i = 1, 3 do
        coroutine.yield(i)
    end
end)

print(co2())  -- 1
print(co2())  -- 2
print(co2())  -- 3
```

**`resume` vs `wrap`**：
- `resume` 返回 `(ok, value)`，第一个值表示是否成功，可传递 yield 参数
- `wrap` 返回函数，调用更简洁，但出错会直接抛出

### 4.2 yield 参数传递

`coroutine.yield(...)` 的参数会作为 `resume` 的返回值返回到调用方；下一次 `resume` 的参数会作为 `yield` 的返回值传递回协程内部。

```lua
local co = coroutine.create(function(a, b)
    print("收到:", a, b)           -- 收到: 1, 2
    local c, d = coroutine.yield(a + b)
    print("再次收到:", c, d)       -- 再次收到: 10, 20
    return a + b + c + d
end)

coroutine.resume(co, 1, 2)          -- 输出"收到: 1, 2"，返回 3
coroutine.resume(co, 10, 20)        -- 输出"再次收到: 10, 20"，返回 33
```

### 4.3 协程状态

```lua
coroutine.status(co)
-- "suspended" — 挂起（创建后或 yield 后）
-- "running"   — 正在运行
-- "normal"    — 被其他协程 resume
-- "dead"      — 已结束或出错
```

---

## 5. 值传递与引用传递

### 5.1 语义区分

Lua 只有一种参数传递语义，但不同类型表现不同：

| 类型分类 | 类型 | 行为 |
|----------|------|------|
| 值类型 | `nil`, `boolean`, `number`, `string` | 赋值/传参时创建副本 |
| 引用类型 | `table`, `function`, `userdata`, `thread` | 赋值/传参时传递引用 |

```lua
-- 值类型：副本独立
local a = 10
local b = a
b = 20
print(a)  -- 10（不受影响）

-- 引用类型：指向同一对象
local t1 = { name = "Lua", version = 5.4 }
local t2 = t1
t2.version = 5.5
print(t1.version)  -- 5.5（受影响）
```

### 5.2 深拷贝与浅拷贝

```lua
-- 浅拷贝：只复制一层
function shallowCopy(original)
    local copy = {}
    for k, v in pairs(original) do
        copy[k] = v
    end
    return copy
end

-- 深拷贝：递归复制所有嵌套 table
function deepCopy(original)
    if type(original) ~= "table" then
        return original
    end
    local copy = {}
    for k, v in pairs(original) do
        copy[k] = deepCopy(v)
    end
    return copy
end

-- 验证
local t1 = { a = 1, b = { x = 10, y = 20 } }
local t2 = shallowCopy(t1)
local t3 = deepCopy(t1)
t2.b.x = 999  -- 影响 t1.b.x
t3.b.x = 888  -- 不影响 t1.b.x
```

---

## 6. 内存管理

### 6.1 垃圾回收

Lua 使用**增量式三色标记-清除** GC：

1. 初始所有对象为**白色**
2. 标记阶段开始：从根集（全局变量、活跃函数的局部变量）可达的对象标记为**灰色**
3. 逐个取出灰色对象，将其可达的白色对象标记为灰色，自身标记为**黑色**
4. 清除阶段：不存在灰色对象时，清除白色对象，将所有黑色对象标记回白色

增量式标记允许 GC 随时暂停和继续，减少停顿。

```lua
collectgarbage("count")   -- 返回当前内存使用量（KB）
collectgarbage("collect") -- 强制执行一次完整 GC
```

### 6.2 弱引用

通过 `__mode` 元字段设置弱引用表，防止循环引用导致的内存泄漏：

| `__mode` 值 | 说明 |
|-------------|------|
| `"k"` | 键为弱引用 |
| `"v"` | 值为弱引用 |
| `"kv"` | 键和值均为弱引用 |

```lua
local t = {}
setmetatable(t, { __mode = "k" })  -- 键为弱引用

local key = { name = "temp" }
t[key] = 1
key = nil        -- 对象只被 t 的键引用
collectgarbage() -- GC 后 t 中对应项被回收
```

---

## 7. 全局变量管理

Lua 所有全局变量存储在 `_G` 表中。`local` 变量不在 `_G` 中。

**防止全局变量泛滥**：通过配置 `_G` 的 `__newindex` 和 `__index` 元方法可以限制全局变量的声明范围。

```lua
-- 多脚本中 local 变量其他脚本不可见
local a = 10     -- 仅当前脚本可见
b = 20           -- 全局变量，存储在 _G["b"]

-- require 返回的模块
local myModule = require("mymodule")
package.loaded["mymodule"] = nil  -- 卸载模块
```

### 随机数

Lua 使用**线性同余法**生成均匀分布的伪随机数序列。`math.randomseed(os.time())` 设置种子初始化序列。

---

## 8. Lua 循环语法

### 8.1 数值 for

```lua
-- 正向遍历
for i = 1, #array do print(array[i]) end

-- 指定步长
for i = 1, 10, 2 do print(i) end  -- 1, 3, 5, 7, 9

-- 反向遍历
for i = #array, 1, -1 do print(array[i]) end
```

### 8.2 while / repeat-until

```lua
while condition do
    -- body
end

repeat
    -- body
until condition  -- 条件为真时退出
```

### 8.3 特殊语法

```lua
-- 多变量赋值
a, b, c = 1, 2, "123"

-- 伪三目运算符（利用短路）
local res = (x > y) and x or y  -- x > y 时取 x，否则取 y

-- 只有 nil 和 false 为假，0 和空字符串均为真
print(0 and 1)  -- 1
```
