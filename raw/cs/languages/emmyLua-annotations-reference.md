---
title: "EmmyLua 注解完全参考"
type: source
updated: 2026-06-02
tags: [lua, emmylua, luals, annotations, type-system, luacats]
description: EmmyLua 注解（Annotation）完全参考手册。涵盖约 25 种注解，分 9 组：类型系统（@class/@field/@alias/@type/@enum/@generic）、函数签名（@param/@return/@overload/@vararg）、异步与约束（@async/@nodiscard）、类型操作（@cast/@as）、模块系统（@module/@meta/@package）、运算符（@operator）、代码质量（@deprecated/@diagnostic）、文档与引用（@see/@version/@source）、IDE 辅助（region/language）
---

# EmmyLua 注解完全参考

EmmyLua 注解（Annotation）是以 `---` 开头的特殊注释，为动态类型 Lua 提供静态类型标注能力。注解被 EmmyLua 语言服务器解析后，驱动 IDE 的智能补全、类型检查、代码导航和文档生成。

> **LuaLS v3.0+ vs 原始 EmmyLua：** Lua Language Server（原 Sumneko）v3.0 起不再与原始 EmmyLua（IntelliJ 插件）注解风格完全兼容。本文以 LuaLS/LuaCATS 规范为准，标注差异处会特别说明。

本文涵盖约 25 种注解，按功能分为 9 组。

---

## 1. 类型系统

### 1.1 `@class` — 类定义

声明一个类或表结构，可在项目全局作为类型使用。

**语法：**

```lua
---@class <类名>[: <父类1>[, <父类2>...]]
---@class (exact) <类名>[: <父类>...]
---@class (partial) <类名>
```

| 修饰符 | 说明 |
|--------|------|
| _(无)_ | 普通类，允许后续添加字段 |
| `(exact)` | 精确类，禁止动态添加新字段 |
| `(partial)` | 部分类，扩展现有类定义 |

**示例：**

```lua
-- 基础类
---@class Person
local Person = {}

-- 继承
---@class Employee : Person
local Employee = {}

-- 多继承（mixin）
---@class GameObject : Transform, EventDispatcher
local GameObject = {}

-- 精确类：添加未声明字段会触发警告
---@class (exact) Point
---@field x number
---@field y number
local Point = {}
Point.x = 1   -- OK
Point.z = 3   -- Warning: 未声明字段 z

-- 泛型类
---@class Array<T> : { [integer]: T }
---@class Dictionary<K, V> : { [K]: V }
```

### 1.2 `@field` — 字段定义

为类声明字段，支持访问控制和索引签名。必须紧跟在 `@class` 之后。

**语法：**

```lua
---@field [<访问控制>] <字段名>[?] <类型> [@描述]
---@field [<访问控制>] [<键类型>] <值类型> [@描述]
```

**访问控制：**

| 修饰符 | 可见范围 |
|--------|----------|
| `public` | 所有位置（默认） |
| `protected` | 类内部及子类 |
| `private` | 仅类内部 |
| `package` | 同包内（v3.6+） |

**示例：**

```lua
---@class UserProfile
---@field public id number 用户ID
---@field public name string 用户名
---@field private m_password string 密码（私有）
---@field avatar? string 头像URL（可选字段）
---@field tags string[] 标签列表

-- 索引签名：任意字符串键
---@class Configuration
---@field host string 主机地址
---@field port number 端口号
---@field [string] any 其他配置项（任意字符串键）

-- 继承中的访问控制
---@class Animal
---@field protected legs integer 腿的数量
---@field private m_id integer 内部ID

---@class Dog : Animal
function Dog:legCount()
    return self.legs    -- OK（protected 可被子类访问）
    -- return self.m_id  -- Error（private 不可被子类访问）
end
```

### 1.3 `@alias` — 类型别名

创建可复用的类型别名，常用于枚举字符串字面量和复杂类型简化。

**语法：**

```lua
-- 简单别名
---@alias <别名> <类型表达式>

-- 多行枚举风格
---@alias <别名>
---| '<值1>' [# 描述1]
---| '<值2>' [# 描述2]
```

**示例：**

```lua
-- 简单别名
---@alias userID integer
---@alias ErrorMessage string

-- 字面量联合类型
---@alias Status "active" | "inactive" | "pending"

---@alias HTTPMethod
---| '"GET"'     # 获取资源
---| '"POST"'    # 创建资源
---| '"PUT"'     # 更新资源
---| '"DELETE"'  # 删除资源

-- 泛型别名
---@alias Pair<T> { first: T, second: T }
---@alias mark<K> { [K]: true }
```

**`@alias` vs `@enum`：** `@alias` 适用于轻量级内联值集合，无需背靠 Lua 表；`@enum` 适用于已有 Lua 表定义的枚举。

### 1.4 `@type` — 变量类型声明

显式声明变量/字段的类型，覆盖类型推断。

**语法：**

```lua
---@type <类型>[|类型...] [@描述]
```

**示例：**

```lua
-- 基础类型
---@type string
local userName = "张三"

---@type number
local age = 25

-- 联合类型
---@type string | number
local id = 1001

-- 可选类型（等价于 string | nil）
---@type string?
local optionalName = nil

-- 数组类型
---@type number[]
local scores = {95, 87, 92}

---@type (string | number)[]
local mixed = {"hello", 42}

-- 字典类型
---@type table<string, number>
local ages = { Alice = 25, Bob = 30 }

-- 表字面量类型
---@type { name: string, age: number, email: string? }
local user = { name = "张三", age = 25 }

-- 嵌套表
---@type { user: { id: number, name: string }, permissions: string[] }
local data = {
    user = { id = 1, name = "张三" },
    permissions = { "read", "write" }
}

-- 函数类型
---@type fun(x: number, y: number): number
local add = function(x, y) return x + y end

---@type fun(name: string, age: number): { name: string, age: number }
local createUser = function(name, age) return { name = name, age = age } end

-- 类类型引用
---@class User
---@field id number
---@field name string

---@type User
local currentUser = { id = 1, name = "张三" }

---@type User[]
local userList = {
    { id = 1, name = "张三" },
    { id = 2, name = "李四" }
}
```

### 1.5 `@enum` — 枚举类型

将 Lua 表标记为枚举，使表的键可作为类型化值使用。

**语法：**

```lua
---@enum <枚举名>         -- 值枚举（使用表的值）
---@enum (key) <枚举名>   -- 键枚举（使用表的键作为枚举值）
```

**示例：**

```lua
-- 数值值枚举
---@enum HTTPStatus
local HTTPStatus = {
    OK = 200,
    NOT_FOUND = 404,
    INTERNAL_ERROR = 500,
    BAD_REQUEST = 400,
    UNAUTHORIZED = 401
}

---@param status HTTPStatus
---@return string
function getStatusMessage(status)
    if status == HTTPStatus.OK then return "请求成功"
    elseif status == HTTPStatus.NOT_FOUND then return "资源未找到"
    else return "未知状态" end
end

-- 字符串值枚举
---@enum LogLevel
local LogLevel = {
    DEBUG = "debug",
    INFO = "info",
    WARN = "warn",
    ERROR = "error"
}

-- 键枚举（使用表的键）
---@enum (key) Permission
local Permission = {
    READ = true,
    WRITE = true,
    DELETE = true,
    ADMIN = true
}

---@param perm Permission
function checkPermission(perm) end
checkPermission("READ")   -- OK
checkPermission("EXECUTE") -- Warning

-- 位标志枚举
---@enum FileMode
local FileMode = {
    READ = 1,      -- 0b0001
    WRITE = 2,     -- 0b0010
    EXECUTE = 4,   -- 0b0100
    DELETE = 8     -- 0b1000
}

---@param mode FileMode
---@param permission FileMode
---@return boolean
function hasPermission(mode, permission)
    return (mode & permission) ~= 0
end
```

### 1.6 `@generic` — 泛型

启用泛型编程，类型参数在调用处被解析为具体类型。可约束泛型继承自某父类型。

**语法：**

```lua
---@generic <T>[: <约束类型>] [, <U>[: <约束类型>]...]
```

**泛型函数（类型从参数值推断）：**

```lua
---@generic T
---@param value T
---@return T
function identity(value) return value end

local s = identity("hello")  -- s: string
local n = identity(42)       -- n: number

---@generic T
---@param array T[]
---@param predicate fun(item: T): boolean
---@return T[]
function filter(array, predicate)
    local result = {}
    for _, item in ipairs(array) do
        if predicate(item) then table.insert(result, item) end
    end
    return result
end

---@generic T, R
---@param array T[]
---@param mapper fun(item: T): R
---@return R[]
function map(array, mapper)
    local result = {}
    for _, item in ipairs(array) do
        table.insert(result, mapper(item))
    end
    return result
end
```

**捕获模式（反引号捕获类型名）：**

```lua
---@generic T
---@param className `T`   -- 捕获字符串对应的类型名
---@return T
function new(className) end

local obj = new("Car")   -- obj: Car
```

**泛型约束：**

```lua
---@generic T : table
---@param obj T
---@return T
function deepClone(obj)
    if type(obj) ~= "table" then return obj end
    local copy = {}
    for k, v in pairs(obj) do copy[k] = deepClone(v) end
    return copy
end
```

**泛型类：**

```lua
---@class List<T>
---@field private items T[]
local List = {}

---@generic T
---@return List<T>
function List.new()
    return setmetatable({ items = {} }, { __index = List })
end

---@param item T
function List:add(item)
    table.insert(self.items, item)
end

---@param index number
---@return T?
function List:get(index)
    return self.items[index]
end

---@generic R
---@param mapper fun(item: T): R
---@return List<R>
function List:map(mapper)
    local result = List.new()
    for _, item in ipairs(self.items) do
        result:add(mapper(item))
    end
    return result
end

-- 使用
---@type List<string>
local strList = List.new()
strList:add("hello")
strList:add("world")
local lenList = strList:map(function(s) return #s end)  -- List<number>
```

---

## 2. 函数签名

### 2.1 `@param` — 参数定义

声明函数参数的类型和描述。

**语法：**

```lua
---@param <参数名>[?] <类型>[|类型...] [@描述]
```

| 标记 | 含义 |
|------|------|
| `name?` | 可选参数（可为 nil） |
| `...` | 可变参数 |

**示例：**

```lua
---@param name string 用户名
---@param age number 年龄
---@param active? boolean 是否激活（可选）
---@param tags string[] 标签列表
---@param options table<string, any>|nil 可选配置表
---@param callback fun(err: string?, data: any): nil 回调函数
function setup(name, age, active, tags, options, callback) end

-- 可变参数
---@param index integer 起始索引
---@param ... string 要添加的标签（可变参数）
function addTags(index, ...) end
```

### 2.2 `@return` — 返回值定义

声明函数返回值类型。

**语法：**

```lua
---@return <类型> [名称 [@描述] | # @描述]
```

**示例：**

```lua
-- 简单返回值
---@return boolean 是否成功
function tryConnect() end

-- 带名称的返回值
---@return boolean success 操作是否成功
---@return string|nil errorMessage 错误信息
function validate(data) end

-- 用 # 描述（不给名称）
---@return boolean # 是否有效
function check() end

-- 多返回值
---@return number x X坐标
---@return number y Y坐标
---@return number z Z坐标
function getPosition() end

-- 可空返回值
---@return string|nil 查找到的用户名
function findUser(id) end
```

### 2.3 `@overload` — 函数重载

为同一函数提供多种调用签名。重载签名不支持单独的描述文字。

**语法：**

```lua
---@overload fun(<参数列表>): <返回值>[, <返回值>...]
```

**示例：**

```lua
-- 不同参数组合
---@param objectID integer 对象ID
---@param whenOutOfView? boolean 是否仅在不可见时移除
---@return boolean
---@overload fun(objectID: integer): boolean
function removeObject(objectID, whenOutOfView) end

-- 构造函数重载
---@class Vector3
---@field x number
---@field y number
---@field z number
---@overload fun(): Vector3                  -- 无参构造
---@overload fun(x: number, y: number): Vector3  -- 2D 构造
---@overload fun(x: number, y: number, z: number): Vector3
function Vector3.new(x, y, z) end
```

### 2.4 `@vararg` — 可变参数类型

标注 `...` 可变参数的类型。

**语法：**

```lua
---@vararg <类型>
```

**示例：**

```lua
---@vararg string
---@return string
function concat(...)
    local parts = { ... }  -- parts: string[]
    return table.concat(parts, ", ")
end
```

---

## 3. 异步与约束

### 3.1 `@async` — 异步函数标记

标记函数为异步。启用 `hint.await` 后，语言服务器会对调用处显示 await 提示。

**语法：**

```lua
---@async
```

**示例：**

```lua
---@async
---@param url string 请求URL
---@return string body 响应体
---@return number statusCode 状态码
function http.get(url) end

---@async
---@param seconds number 等待秒数
function waitForSeconds(seconds) end
```

### 3.2 `@nodiscard` — 禁止忽略返回值

标记函数的返回值不可被丢弃。调用方未捕获返回值时产生警告。

**语法：**

```lua
---@nodiscard
```

**示例：**

```lua
---@return string userId
---@nodiscard
function getCurrentUserId() end

getCurrentUserId()  -- Warning: 忽略了不可丢弃的返回值

-- 正确的用法
local userId = getCurrentUserId()

---@return boolean success
---@return string|nil errorMessage
---@nodiscard
function tryConnect(host) end
```

---

## 4. 类型操作

### 4.1 `@cast` — 变量类型转换

对已声明变量进行类型添加/移除操作。

**语法：**

```lua
---@cast <变量名> [+|-]<类型表达式>[, [+|-]<类型表达式>...]
```

| 操作符 | 含义 |
|--------|------|
| 无前缀 | 覆盖为指定类型 |
| `+` | 添加到联合类型 |
| `-` | 从联合类型中移除 |
| `+?` | 允许为 nil |

**示例：**

```lua
---@type integer | string
local x

---@cast x string
-- x: string（覆盖）

---@type integer
local y

---@cast y +boolean
-- y: integer | boolean

---@type integer | string
local z

---@cast z -integer
-- z: string

---@cast x +?   -- x 现在允许 nil：x: string?
```

### 4.2 `@as` — 表达式类型断言

在表达式位置强制转换类型，适用于函数参数内联转换。

**语法：**

```lua
---@as <类型>
```

**示例：**

```lua
local x = nil

---@param key string
local function doSomething(key) end

-- 将 x 断言为 string 后传入
doSomething(x ---@as string)

-- 类型缩小后的断言
---@type number|nil
local value = getValue()
local result = math.floor(value ---@as number)
```

---

## 5. 模块系统

### 5.1 `@module` — 模块声明

告诉语言服务器一个变量按 `require` 语义处理，提供外部模块的类型信息。

**语法：**

```lua
---@module '<模块名>'
```

**示例：**

```lua
-- 等价于 local http = require 'http'
---@module 'http'
local http
```

在定义文件中，`@module` 可用于声明文件本身的模块身份。

### 5.2 `@meta` — 元定义文件

标记文件为纯类型定义文件——其中的 Lua 代码仅用于类型标注，不会被实际执行。

**语法：**

```lua
---@meta [名称]
```

| 名称 | 行为 |
|------|------|
| 已命名字符串 | 仅可通过 `require '<名称>'` 加载 |
| `_` | 不可被 require（私有定义） |
| 省略 | 标记为 meta 但无名称限制 |

**meta 文件的行为变化：**
- 补全不显示 meta 文件中的上下文
- Hover 在 require 上显示 `[meta]` 而非绝对路径
- 查找引用跳过 meta 文件内部结果

**示例：**

```lua
---@meta mylibrary

---@class MyClass
---@field x number
---@field y number

function MyClass:getPosition() end
```

### 5.3 `@package` — 文件私有

标记函数为文件级私有，仅本文件内可访问。

**语法：**

```lua
---@package
```

**示例：**

```lua
---@package
function MyClass:internalHelper()
    -- 仅本文件内可调用
end
```

---

## 6. 运算符

### 6.1 `@operator` — 运算符元方法类型

为运算符元方法（`__add`, `__sub`, `__unm`, `__call` 等）声明类型签名，使语言服务器能推断运算符结果的类型。

**语法：**

```lua
---@operator <操作>[(参数类型)]: <返回类型>
```

**操作符映射：**

| 注解操作 | 元方法 | Lua 表达式 |
|----------|--------|------------|
| `add` | `__add` | `a + b` |
| `sub` | `__sub` | `a - b` |
| `mul` | `__mul` | `a * b` |
| `div` | `__div` | `a / b` |
| `mod` | `__mod` | `a % b` |
| `pow` | `__pow` | `a ^ b` |
| `unm` | `__unm` | `-a` |
| `concat` | `__concat` | `a .. b` |
| `eq` | `__eq` | `a == b` |
| `lt` | `__lt` | `a < b` |
| `le` | `__le` | `a <= b` |
| `call` | `__call` | `a()` |
| `len` | `__len` | `#a` |
| `index` | `__index` | `a[key]` |
| `newindex` | `__newindex` | `a[key] = v` |

**示例：**

```lua
---@class Vector
---@field x number
---@field y number
---@operator add(Vector): Vector
---@operator sub(Vector): Vector
---@operator unm: Vector
local Vector = {}

---@type Vector
local v1, v2
local v3 = v1 + v2   -- v3: Vector
local v4 = v1 - v2   -- v4: Vector
local v5 = -v1       -- v5: Vector

-- __call 运算符
---@class Signal
---@operator call(any): nil
```

---

## 7. 代码质量

### 7.1 `@deprecated` — 弃用标记

标记函数/字段为已弃用，调用处产生 `deprecated` 诊断并在 IDE 中显示删除线。

**语法：**

```lua
---@deprecated
```

**示例：**

```lua
---@deprecated 请使用 newMethod() 替代
function oldMethod() end

oldMethod()  -- 显示为 ~~oldMethod~~，触发弃用警告
```

### 7.2 `@diagnostic` — 诊断控制

控制特定行的诊断规则开启/关闭。

**语法：**

```lua
---@diagnostic <操作>:<规则>[, <规则>...]
```

| 操作 | 作用范围 |
|------|----------|
| `disable-next-line` | 仅禁用下一行 |
| `disable-line` | 禁用当前行 |
| `disable` | 整个文件禁用 |
| `enable` | 整个文件启用 |

**示例：**

```lua
-- 禁用下一行的 unused-local 警告
---@diagnostic disable-next-line: unused-local
local tempVar = calculateSomething()

-- 禁用下一行的多个规则
---@diagnostic disable-next-line: unused-local, undefined-global
local x = UnknownGlobal

-- 文件级禁用
---@diagnostic disable: spell-check
```

常见诊断规则：`unused-local`, `undefined-global`, `redundant-parameter`, `deprecated`, `spell-check`, `await`.

---

## 8. 文档与引用

### 8.1 `@see` — 交叉引用

标注对其他符号的引用。

**语法：**

```lua
---@see <引用路径> [@描述]
```

**示例：**

```lua
---@see Person#getName @查看姓名获取方法
---@see HTTPStatus @HTTP状态码枚举
function getStatus() end
```

### 8.2 `@version` — 版本要求

标记函数的 Lua 版本要求。

**语法：**

```lua
---@version <版本约束>
```

**示例：**

```lua
---@version >5.2, JIT
function bitwiseOp(a, b) end  -- 需要 Lua 5.3+ 或 LuaJIT
```

### 8.3 `@source` — 源代码引用

标注代码片段的来源。

**语法：**

```lua
---@source <URL或路径>
```

**示例：**

```lua
---@source https://github.com/example/lib/blob/main/utils.lua
function safeDivide(a, b) end
```

---

## 9. IDE 辅助

### 9.1 代码折叠区域

Lua 语言服务器支持的两种代码折叠标记：

```lua
-- Region 风格（推荐）
--region 初始化逻辑
local function init()
    -- ...
end
--endregion

-- 花括号风格
--{{{ 私有工具函数
local function helper1() end
local function helper2() end
--}}}
```

两种风格效果相同：将包裹的代码块折叠为一个可展开区域。

### 9.2 `--- @language` — 内嵌语言

标记字符串包含某种语言的嵌入代码，启用对应语言的语法高亮和检查。

**语法：**

```lua
---@language <语言标识>
```

**示例：**

```lua
---@language JSON
local config = [[
{
    "name": "MyApp",
    "version": "1.0.0",
    "debug": false
}
]]

---@language SQL
local query = [[
SELECT id, name, email
FROM users
WHERE active = 1
ORDER BY name ASC
]]

---@language XML
local html = [[
<div class="container">
    <h1>Title</h1>
</div>
]]
```

---

## 最佳实践总结

1. **注解顺序** — `@field` 紧跟在 `@class` 之后；`@param`/`@return` 紧贴在函数之前
2. **选择合适的枚举方式** — 轻量级字面量用 `@alias`，已有 Lua 表用 `@enum`
3. **泛型捕获** — 需要捕获类型名称（而非值的类型）时用反引号 `` `T` ``
4. **访问控制** — 合理使用 `public`/`private`/`protected` 模拟面向对象封装
5. **Markdown 文档** — 注解描述支持 Markdown 格式，可嵌入代码块和列表
6. **全局变量管理** — 在 `.emmyrc.json` 的 `diagnostics.globals` 中配置全局变量白名单，不要滥用 `@diagnostic disable`
7. **meta 文件** — 第三方库的类型存根文件加 `---@meta` 标记，避免污染补全上下文

## 参考资源

| 资源 | 链接 |
|------|------|
| LuaLS 官方注解文档 | https://luals.github.io/wiki/annotations/ |
| EmmyLua Annotations (GitHub Wiki) | https://github.com/LuaLS/lua-language-server/wiki/Annotations |
| EmmyLua 注解中文文档 | https://emmylua.github.io/zh_CN/annotation.html |
