---
title: "XLua 互调与热补丁"
type: source
updated: 2026-06-02
tags: [unity, xlua, lua, csharp, hotfix, il-injection]
description: XLua 互调与热补丁完整参考：C# 调用 Lua（LuaEnv/自定义 Loader/Table 映射/Function 映射/By Value vs By Reference）、Lua 调用 C#（CS 命名空间/ref out 参数/泛型容器/Delegate Event）、性能优化（generic 方法缓存/Generate 代码生成/数组优化）、IL 注入热补丁原理与配置
---

# XLua 互调与热补丁

XLua 是腾讯开源的 Unity Lua 编程方案，核心能力包括 C# 与 Lua 双向互调，以及基于 IL 注入的热补丁机制。

---

## 1. C# 调用 Lua

### 1.1 LuaEnv 生命周期

```csharp
LuaEnv luaEnv = new LuaEnv();     // 创建虚拟机，推荐全局唯一

luaEnv.DoString("print('hello')");
luaEnv.DoString("require('Main')");

luaEnv.Tick();     // 垃圾回收，建议按帧或按间隔调用
luaEnv.Dispose();  // 销毁虚拟机
```

`DoString("require 'xxx'")` 默认从 `Resources` 加载 `.lua.txt` 文件。可通过 `AddLoader` 自定义加载逻辑。

### 1.2 自定义 Loader

```csharp
// 文件系统 Loader
private byte[] FileLoader(ref string filepath) {
    string path = Application.dataPath + "/Lua/" + filepath + ".lua";
    if (File.Exists(path))
        return File.ReadAllBytes(path);
    return null;
}

// AssetBundle Loader
private byte[] ABLoader(ref string filepath) {
    TextAsset asset = ABManager.Instance.LoadRes<TextAsset>("lua", filepath + ".lua");
    return asset?.bytes;
}

// 注册
luaEnv.AddLoader(FileLoader);
luaEnv.AddLoader(ABLoader);
```

Loader 按注册顺序依次尝试，返回非 null 的 byte 数组即表示加载成功。Lua 内部 `require` 也会自动经过这些 Loader。

### 1.3 获取全局基本类型

```csharp
int a = luaEnv.Global.Get<int>("a");
string b = luaEnv.Global.Get<string>("b");
bool c = luaEnv.Global.Get<bool>("c");

luaEnv.Global.Set("testNum", 1000);  // 写入
```

> 注意：`Get<T>` 对基本类型是**值拷贝**，修改 C# 变量不会影响 Lua 侧。

### 1.4 Table 映射

| 映射方式 | 机制 | 性能 | 类型安全 | 推荐度 |
|----------|------|------|----------|--------|
| `class` / `struct` | 值拷贝，字段名对应 | 中等 | 是 | 只读场景 |
| `interface` | 引用拷贝，需 `[CSharpCallLua]` | 高 | 是 | 需要双向同步 |
| `Dictionary<,>` / `List<>` | 值拷贝 | 中等 | 是 | 简单 key-value / 数组 |
| `LuaTable` | 引用，无代码生成 | 慢 ~10x | 否 | 仅原型阶段 |

**映射到 class（by value）**：

```csharp
public class CallLuaClass {
    public int testInt;
    public bool testBool;
    public float testFloat;
    public string testString;
    public UnityAction func;  // Lua 函数可映射到委托
}

CallLuaClass obj = luaEnv.Global.Get<CallLuaClass>("testClass");
// 字段赋值是值拷贝，修改 C# 对象不影响 Lua 原 table
```

**映射到 interface（by reference）**：

```csharp
[CSharpCallLua]
public interface ItfD {
    int f1 { get; set; }
    int f2 { get; set; }
    int add(int a, int b);
}

ItfD d = luaEnv.Global.Get<ItfD>("d");
d.f1 = 100;  // 直接修改 Lua 原 table
```

interface 是引用拷贝，get/set 会实时访问 Lua table。必须将 interface 加入 `[CSharpCallLua]` 生成列表。

**映射到 LuaTable（by ref，不推荐）**：

```csharp
LuaTable table = luaEnv.Global.Get<LuaTable>("testClass");
Debug.Log(table.Get<int>("testInt"));
// 需要手动释放
table.Dispose();
```

### 1.5 Function 映射

**映射到 Delegate（推荐）**：

```csharp
[CSharpCallLua]
public delegate int D1(int a);

[CSharpCallLua]
public delegate int D3(int a, out bool c, out int d);  // 多返回值

[CSharpCallLua]
public delegate void D4(int a, params object[] args);  // 变长参数

D1 fn = luaEnv.Global.Get<D1>("testFun2");
int result = fn(100);
```

多返回值映射规则：
- 返回值、`out` 参数、`ref` 参数从左往右对应 Lua 的多返回值
- `out` 参数不需要初始化，`ref` 参数需要传入默认值

```csharp
// Lua: function testFun3(a) return 100, true, 200 end
D3 fn = luaEnv.Global.Get<D3>("testFun3");
int c; bool ss;
int ret = fn(100, out ss, out c);
// ret = 100, ss = true, c = 200
```

**映射到 LuaFunction（不推荐）**：

```csharp
LuaFunction lf = luaEnv.Global.Get<LuaFunction>("testFun2");
object[] results = lf.Call(30);
// results[0] = 返回值，类型为 object
```

### 1.6 By Value vs By Reference

| 特性维度 | By Value | By Reference |
|----------|----------|--------------|
| 传递机制 | 创建独立副本 | 传递原始数据引用 |
| 数据同步 | 不影响原始数据 | 直接修改原始数据 |
| 性能 | 轻量 | 较慢 |
| 类型安全 | C# 泛型保证 | `object[]` 不安全 |
| 示例 | `class`, `Dictionary`, `List`, `LuaFunction` | `interface`, `LuaTable`, `delegate` |

**最佳实践**：
1. 访问 Lua 全局数据（尤其是 table/function）代价较大，建议在初始化时获取一次并缓存
2. Lua 侧尽量以 delegate + interface 暴露，让调用方与 XLua 解耦

---

## 2. Lua 调用 C#

### 2.1 基础调用

所有 C# 类型通过 `CS` 全局命名空间访问：

```lua
-- 创建对象（无需 new 关键字）
local obj = CS.UnityEngine.GameObject()
local obj2 = CS.UnityEngine.GameObject("hello")

-- 使用别名提高性能
local GameObject = CS.UnityEngine.GameObject
local obj3 = GameObject.Find("MainCamera")

-- 静态属性
local dt = CS.UnityEngine.Time.deltaTime
CS.UnityEngine.Time.timeScale = 0.5

-- 成员方法（必须用冒号）
obj.transform:Translate(CS.UnityEngine.Vector3.right)
```

### 2.2 ref / out 参数

C# 的 `ref` / `out` 在 Lua 侧以多返回值形式体现：

```lua
-- C#: int RefFun(int a, ref int b, ref int c, int d)
-- 返回顺序: 返回值, ref参数...
-- ref 参数需要传入占位值
local a, b, c = obj:RefFun(1, 0, 0, 1)

-- C#: int OutFun(int a, out int b, out int c, int d)
-- out 参数不需要占位
local a, b, c = obj:OutFun(20, 30)

-- 混合: int RefOutFun(int a, out int b, ref int c, int d)
local a, b, c = obj:RefOutFun(20, 1)
```

### 2.3 枚举

```lua
-- 直接使用
local cube = GameObject.CreatePrimitive(CS.UnityEngine.PrimitiveType.Cube)

-- 枚举转换
CS.E_MyEnum.__CastFrom(1)        -- 从整数转换
CS.E_MyEnum.__CastFrom("ATK")    -- 从字符串转换
```

### 2.4 泛型容器

Lua 中没有泛型语法。泛型类型需通过构造函数形式获取类型实例：

```lua
-- List
local List_String = CS.System.Collections.Generic.List(CS.System.String)
local list = List_String()
list:Add("hello")

-- Dictionary
local Dic_String_Vector3 = CS.System.Collections.Generic.Dictionary(
    CS.System.String, CS.UnityEngine.Vector3)
local dic = Dic_String_Vector3()
dic:Add("right", CS.UnityEngine.Vector3.right)

-- Dictionary 访问需使用特殊方法（不能直接 dic["key"]）
local val = dic:get_Item("right")
dic:set_Item("right", newValue)
local ok, val = dic:TryGetValue("right")

-- 数组
local arr = CS.System.Array.CreateInstance(
    typeof(CS.System.Int32), 10)
arr[0] = 42
```

### 2.5 Delegate 与 Event

```lua
-- Delegate
obj.del = luaFunc
obj.del = obj.del + anotherFunc  -- 添加
obj.del = nil                      -- 清空
obj.del()                          -- 调用

-- Event（不同于 delegate）
obj:eventAction("+", luaCallback)   -- 添加
obj:eventAction("-", luaCallback)   -- 移除
obj:DoEvent()                       -- 触发
obj:ClearEvent()                    -- 清空（需 C# 侧暴露方法）
```

### 2.6 扩展方法与重载

扩展方法必须添加 `[LuaCallCSharp]` 特性才能从 Lua 访问：

```lua
-- C#: public static void Move(this Func obj)
obj:Move()  -- 扩展方法调用
```

重载函数：由于 Lua 只有 number 一种数值类型，多精度重载支持不佳。推荐使用反射：

```lua
local m1 = typeof(CS.Class):GetMethod("Calc", { typeof(CS.System.Int32) })
local m2 = typeof(CS.Class):GetMethod("Calc", { typeof(CS.System.Single) })
local f1 = xlua.tofunction(m1)
```

### 2.7 类型转换

```lua
-- typeof
typeof(CS.UnityEngine.ParticleSystem)

-- 强制指定生成代码访问（提高性能）
cast(obj, typeof(CS.Tutorial.Calc))

-- null 判断（nil ≠ null）
function IsNull(obj)
    return obj == nil or obj:Equals(nil)
end
```

---

## 3. 性能优化

### 3.1 对象访问开销

C# 对象不能直接作为指针给 Lua 操作，XLua 通过 **id → ObjectTranslator 字典** 映射机制：

```
Lua userdata (id) → ObjectTranslator.TryGetValue(id) → C# object
```

**每次使用** object（参数传递、成员访问）都涉及字典查找。因此：

- **持有常用对象引用**：避免临时 userdata 被 GC 后反复创建
- **避免串联 `.` 访问**：如 `transform.position`，每次 `.` 都触发查找
- **使用静态方法**：减少 object 查找，静态方法可直接调用

```csharp
// 推荐：用静态工具方法减少 object 传递
class LuaUtil {
    static void SetPos(GameObject obj, float x, float y, float z) {
        obj.transform.position = new Vector3(x, y, z);
    }
}
```

```lua
-- 在 Lua 中调用静态方法，避免多次 transform 查找
LuaUtil.SetPos(obj, 1, 2, 3)
```

### 3.2 Vector3 / Quaternion 的性能陷阱

Unity 值类型（`Vector3`, `Quaternion`）在 Lua↔C# 间传递开销极大：

1. C# 取出 x, y, z
2. 逐个 push 到 Lua 栈（3 次 push）
3. 构建 Lua table `{ x=0, y=0, z=0 }`
4. 3 次表插入 + 表内存分配

**推荐方案**：使用三个独立的 float 参数替代：

```csharp
// 差
void SetPos(GameObject obj, Vector3 pos);
// 好
void SetPos(GameObject obj, float x, float y, float z);
```

传参避免类型：
- **最严重**：`Vector3`, `Quaternion` 等 Unity 值类型、数组
- **次严重**：`bool`, `string`（Non-Blittable 类型涉及内存转换）、各种 object
- **推荐**：`int`, `float`, `double`

### 3.3 GCOptimize

```csharp
[GCOptimize]
public struct MyStruct { public int x; public int y; }
```

加上 `[GCOptimize]` 的值类型在 Lua↔C# 传递时**不产生 GC alloc**。默认将值传递为 `object`（会装箱），优化后使用指定类型直接传递。

### 3.4 `[LuaCallCSharp]` 与 `[CSharpCallLua]`

- **`[LuaCallCSharp]`**：生成适配代码，避免反射访问。Lua 中调用的所有 C# 类型建议添加。
- **`[CSharpCallLua]`**：生成 delegate/interface 适配代码。所有映射到 delegate 或 interface 的类型必须添加。
- **`[ReflectionUse]`**：阻止 il2cpp 代码剪裁，确保不被意外移除。

```csharp
// 对无法修改源码的第三方类型，使用静态列表
[CSharpCallLua]
public static List<Type> cSharpCallLuaList = new List<Type>() {
    typeof(UnityAction<float>)
};

[LuaCallCSharp]
public static List<Type> luaCallCSharpList = new List<Type>() {
    typeof(GameObject)
};
```

### 3.5 内存泄漏

Lua 持有的 C# 对象通过 `userdata(id) → ObjectTranslator dictionary → C# object` 关联。**只要 Lua 中的 userdata 未被 GC，Dictionary 就持有 C# 对象的引用**，导致 C# 对象无法被 GC 回收。

常见场景：Component 被 Lua 引用后即使 Destroy，仍残留在 Mono 堆中。

**解决**：遍历 ObjectTranslator 的 Dictionary，手动将对应引用置 nil。

---

## 4. 热补丁原理

### 4.1 热补丁流程

1. 添加 `HOTFIX_ENABLE` 宏（Build Settings → Scripting Define Symbols）
2. 对可能变动的类打上 `[Hotfix]` 标签
3. 执行 "XLua/Generate Code" 生成代码
4. 执行 "XLua/Hotfix Inject In Editor" 进行 IL 注入
5. 在 Lua 中调用 `xlua.hotfix()` 注入补丁

### 4.2 代码生成阶段

Generate Code 在 `Gen` 目录下生成 `DelegatesGensBridge.cs`，其中 `DelegateBridge` 类的 `__Gen_Delegate_Imp` 方法负责桥接到 Lua 热补丁函数：

```csharp
namespace XLua {
    public partial class DelegateBridge : DelegateBridgeBase {
        public void __Gen_Delegate_Imp0(object p0) {
            RealStatePtr L = luaEnv.rawL;
            int errFunc = LuaAPI.pcall_prepare(L, errorFuncRef, luaReference);
            // luaReference 指向 xlua.hotfix(CS.XXX, "Start", function(self))
            ObjectTranslator translator = luaEnv.translator;
            translator.PushAny(L, p0);
            PCall(L, 1, 0, errFunc);
            LuaAPI.lua_settop(L, errFunc - 1);
        }
    }
}
```

### 4.3 IL 注入阶段

Hotfix Inject 使用 **Mono.Cecil** 库对程序集进行 IL 注入。注入后，每个标记了 `[Hotfix]` 的方法会生成对应的 `DelegateBridge` 静态变量：

```csharp
[Hotfix(HotfixFlag.Stateless)]
public class Test : MonoBehaviour {
    private static DelegateBridge _c__Hotfix0_ctor;
    private static DelegateBridge __Hotfix0_Start;
    private static DelegateBridge __Hotfix0_Update;

    public Test() : this() {
        _c__Hotfix0_ctor?.__Gen_Delegate_Imp0(this);  // 注入的 hook
    }

    private void Start() {
        if (__Hotfix0_Start != null) {
            __Hotfix0_Start.__Gen_Delegate_Imp0(this);  // 执行热补丁
        } else {
            Debug.Log("test");  // 原有逻辑
        }
    }

    private void Update() {
        __Hotfix0_Update?.__Gen_Delegate_Imp0(this);
    }
}
```

**核心机制**：如果 `xlua.hotfix()` 设置了对应的函数，`DelegateBridge` 非空，则执行 Lua 逻辑；否则回退到原始 C# 逻辑。本质上是在运行时用 Lua 函数替换原本的 C# 方法体。

### 4.4 热补丁 API

**基本用法**：

```lua
-- 替换单个方法
xlua.hotfix(CS.Hotfix, "Add", function(self, a, b)
    return a + b
end)

-- 静态方法不需要 self
xlua.hotfix(CS.Hotfix, "Speak", function(str)
    print(str)
end)

-- 批量替换：传入 table
xlua.hotfix(CS.Hotfix, {
    Update = function(self) print(os.time()) end,
    Add = function(self, a, b) return (a + b) * 100 end,
    Speak = function(str) print(str) end
})
```

**构造函数**（`method_name = ".ctor"`）：

```lua
xlua.hotfix(CS.Hotfix_T, {
    [".ctor"] = function()
        print("Lua 热补丁构造函数")
    end
})
-- 注意：构造函数的补丁不是替换，而是先执行原有逻辑再调用 Lua
```

**属性**：

```lua
-- getter: get_属性名, setter: set_属性名
xlua.hotfix(CS.HotfixTest, {
    set_Age = function(self, v) print("set:", v) end,
    get_Age = function(self) return 100 end
})
```

**索引器**：

```lua
-- set_Item, get_Item
xlua.hotfix(CS.HotfixTest, {
    set_Item = function(self, k, v) print("set", k, v) end,
    get_Item = function(self, k) return 1024 end
})
```

**事件**：

```lua
xlua.hotfix(CS.HotfixTest, {
    add_myEvent = function(self, del) print("add event") end,
    remove_myEvent = function(self, del) print("remove event") end,
})
```

**析构函数**：

```lua
xlua.hotfix(CS.Hotfix_T, {
    Finalize = function(self) print("Finalize") end
})
-- 析构函数的补丁不是替换，而是先调用 Lua 再继续原有逻辑
```

### 4.5 泛型类热补丁

每个泛型实例化是独立类型，需分别打补丁：

```lua
-- 只替换 HotfixTest2<string>，不影响 HotfixTest2<int>
xlua.hotfix(CS.HotfixTest2(CS.System.String), {
    test = function(self, str)
        print("hotfix: " .. str)
    end
})
```

### 4.6 协程替换

```lua
local util = require("xlua.util")

xlua.hotfix(CS.HotFixSubClass, {
    Start = function(self)
        return util.cs_generator(function()
            while true do
                coroutine.yield(CS.UnityEngine.WaitForSeconds(3))
                print("Wait for 3 seconds")
            end
        end)
    end
})
```

`util.cs_generator` 将 Lua 函数包装为 `IEnumerator`，在 Lua 协程中使用 `coroutine.yield` 模拟 C# 的 `yield return`。

### 4.7 调用原始实现

```lua
xlua.hotfix(CS.BaseTest, "Foo", function(self, p)
    print("before base")
    base(self):Foo(p)  -- 调用父类原始实现
    print("after base")
end)
```

`base(csobj)` 返回一个新对象，通过它调用方法会执行原始 C# 逻辑，常用于在补丁前后添加额外行为。

---

## 5. XLua 配置体系

### 5.1 配置方式

xLua 所有配置支持三种方式：打标签、静态列表、动态列表。配置必须放在静态类中，推荐放 Editor 目录。

| 配置项 | 作用 |
|--------|------|
| `[LuaCallCSharp]` | 生成适配代码使 Lua 可高性能访问该类 |
| `[CSharpCallLua]` | 生成 delegate/interface 适配代码 |
| `[Hotfix]` | 标记可热补丁的类 |
| `[GCOptimize]` | 值类型无 GC 优化 |
| `[ReflectionUse]` | 生成 link.xml 阻止 il2cpp 剪裁 |
| `[BlackList]` | 排除不生成适配代码的成员 |
| `[DoNotGen]` | 延迟 wrap（比 ReflectionUse 更细粒度） |
| `[AdditionalProperties]` | GCOptimize 扩展，指定属性名 |

### 5.2 配置建议

1. 对所有较大可能变动的类型加上 `[Hotfix]`
2. 用反射找出所有 delegate 类型，标注 `[CSharpCallLua]`
3. Lua 补丁中需高性能访问的 C# 类型，加上 `[LuaCallCSharp]`
4. 可能被 il2cpp 剪裁的 API 类型，加 `[LuaCallCSharp]` 或 `[ReflectionUse]`

---

## 6. 调试

XLua 调试推荐使用 VS Code 的 Attach 模式（Rider EmmyLua Debug 存在已知 bug）：

1. 安装 VS Code EmmyLua 插件
2. Unity 启动游戏时加载 EmmyLua Debug 模块
3. VS Code 中配置 `.vscode/launch.json` 指定 host/port
4. 在 Lua 中设置断点，F5 Attach

---

## 参考资源

- [xLua GitHub](https://github.com/Tencent/xLua)
- [xLua 官方教程](https://github.com/Tencent/xLua/blob/master/Assets/XLua/Doc/XLua教程.md)
- [xLua Hotfix 文档](https://github.com/Tencent/xLua/blob/master/Assets/XLua/Doc/hotfix.md)

---

## 6. LuaBehaviour 生产模式

LuaBehaviour 是 XLua 项目中最常用的 MonoBehaviour 集成模式——通过一个 C# 组件桥接 Lua 脚本，实现用 Lua 编写游戏逻辑。每个 LuaBehaviour 实例拥有独立的沙箱脚本域（sandbox table），防止全局变量冲突。

```csharp
using UnityEngine;
using XLua;
using System;

namespace XLuaTest
{
    [System.Serializable]
    public class Injection
    {
        public string name;
        public GameObject value;
    }

    [LuaCallCSharp]
    public class LuaBehaviour : MonoBehaviour
    {
        public TextAsset luaScript;
        public Injection[] injections;

        internal static LuaEnv luaEnv = new LuaEnv(); // 所有 LuaBehaviour 共享一个 LuaEnv
        internal static float lastGCTime = 0;
        internal const float GCInterval = 1; // 每秒 GC Tick 一次

        private Action luaStart;
        private Action luaUpdate;
        private Action luaOnDestroy;
        private LuaTable scriptScopeTable;

        void Awake()
        {
            // 为每个脚本创建独立的沙箱域
            scriptScopeTable = luaEnv.NewTable();

            // 设置元表 __index 指向 Global，使沙箱能访问全局变量
            using (LuaTable meta = luaEnv.NewTable())
            {
                meta.Set("__index", luaEnv.Global);
                scriptScopeTable.SetMetaTable(meta);
            }

            // 将 C# 对象注入 Lua 脚本域
            scriptScopeTable.Set("self", this);
            foreach (var injection in injections)
            {
                scriptScopeTable.Set(injection.name, injection.value);
            }

            // 在沙箱域中执行 Lua 脚本
            luaEnv.DoString(luaScript.text, luaScript.name, scriptScopeTable);

            // 从沙箱域中获取 Lua 函数
            Action luaAwake = scriptScopeTable.Get<Action>("awake");
            scriptScopeTable.Get("start", out luaStart);
            scriptScopeTable.Get("update", out luaUpdate);
            scriptScopeTable.Get("ondestroy", out luaOnDestroy);

            luaAwake?.Invoke();
        }

        void Start() => luaStart?.Invoke();

        void Update()
        {
            luaUpdate?.Invoke();

            // 定时触发 Lua GC
            if (Time.time - lastGCTime > GCInterval)
            {
                luaEnv.Tick();
                lastGCTime = Time.time;
            }
        }

        void OnDestroy()
        {
            luaOnDestroy?.Invoke();
            scriptScopeTable.Dispose();
            luaOnDestroy = null;
            luaUpdate = null;
            luaStart = null;
        }
    }
}
```

> [!tip] 沙箱域隔离
> 每个 LuaBehaviour 实例通过 `luaEnv.NewTable()` 创建独立的脚本域（sandbox table），脚本内通过 `self` 访问宿主 GameObject。`Injection` 数组支持在 Inspector 中直接拖拽赋值，将 C# 对象注入 Lua 环境。

> [!note] EmmyLua 调试
> 将 Lua 脚本后缀设为 `.lua.txt` 可被 Unity 识别为 TextAsset。Rider 配合 EmmyLua 插件可实现断点调试，但需将部分代码包装到特定组件中。

## 7. 热补丁高级场景

### 7.1 整个类替换与状态保持

当需要替换一个类的全部方法时，使用 `util.state()` 在 Lua 侧创建持久状态表：

```lua
xlua.hotfix(CS.StatefullTest, {
    ['.ctor'] = function(csobj)
        return util.state(csobj, {evt = {}, start = 0, prop = 0})
    end;
    set_AProp = function(self, v)
        print('set_AProp', v)
        self.prop = v
    end;
    get_AProp = function(self) return self.prop end;
    Start = function(self)
        for _, cb in ipairs(self.evt) do cb(self.start, 2) end
        self.start = self.start + 1
    end;
})
```

`util.state(csobj, initialTable)` 为 C# 对象附加 Lua 状态表，使其在多次 hotfix 调用间保持数据。

### 7.2 操作符重载热补丁

C# 的操作符有内部映射名称：

| C# 操作符 | 内部名称 |
|:----------|:---------|
| `+` | `op_Addition` |
| `-` | `op_Subtraction` |
| `*` | `op_Multiply` |
| `/` | `op_Division` |
| `==` | `op_Equality` |

```lua
xlua.hotfix(CS.MyClass, 'op_Addition', function(self, a, b)
    return a.Value + b.Value
end)
```

### 7.3 事件直接触发

通过 `xlua.private_accessible` 可访问事件的私有 delegate 字段，直接触发事件：

```lua
xlua.private_accessible(CS.MyClass)
-- 直接触发事件
obj['&MyEvent']()
```

> [!warning] 版本差异
> XLua 2.1.11+ 不再需要显式调用 `xlua.private_accessible` 即可直接访问事件字段。

### 7.4 Unity 协程热补丁

使用 `util.cs_generator` 模拟 C# 的 `IEnumerator`：

```lua
local util = require 'xlua.util'
xlua.hotfix(CS.HotFixSubClass, {
    Start = function(self)
        return util.cs_generator(function()
            while true do
                coroutine.yield(CS.UnityEngine.WaitForSeconds(3))
                print('Wait for 3 seconds')
            end
        end)
    end;
})
```

### 7.5 `util.hotfix_ex` 增强热补丁

`util.hotfix_ex` 是 `xlua.hotfix` 的增强版本，允许在 fix 函数内部执行原始 C# 逻辑：

```lua
util.hotfix_ex(CS.MyClass, 'MyMethod', function(self)
    -- 执行原始 C# 逻辑
    -- 缺点是 fix 函数执行略慢
end)
```

### 7.6 `base()` 调用父类实现

子类 override 函数可通过 `base()` 调用父类原始实现：

```lua
xlua.hotfix(CS.BaseTest, 'Foo', function(self, p)
    print('patched BaseTest', p)
    base(self):Foo(p)  -- 调用原始 C# 实现
end)
```
