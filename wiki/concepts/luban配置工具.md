---
title: "Luban 配置工具"
type: concept
updated: 2026-06-03
tags: [unity, luban, config, data-table, tooling, 面试]
---

# Luban 配置工具

Luban 是 focus-creative-games 开源的游戏配置表解决方案，解决游戏配置表的三大痛点：**数据类型安全**（编译期类型检查）、**多格式输出**（同一套 Excel 输出 JSON/Binary/Lua/C#）、**策划友好**（Excel 编辑 + 实时错误提示）。

## 安装与目录结构

通过 git URL 在 Unity Package Manager 安装 `luban_unity` 包，下载工具链后配置 `gen_config.bat`：

```
Assets/
├─ Editor/Luban/gen_config.bat    // 配置生成脚本
├─ ResExcel/                       // Excel 源文件
└─ Generated/Tables/               // 生成的 C# 类型
```

## 运行时加载

通过回调函数控制加载逻辑，支持 Resources、AssetBundle、网络下载等：

```csharp
Tables tables = new Tables(LoadTable);
private JArray LoadTable(string tableName) {
    var asset = Resources.Load<TextAsset>($"ResExcel/JsonData/{tableName}");
    return JArray.Parse(asset.text);
}
var itemCfg = tables.TbItem.Get(1001);
```

> [!warning] `Tables` 构造函数会触发所有关联表的加载，确保回调能处理所有表名。

## 数据类型系统

| 类型 | 说明 | Excel 示例 |
|:-----|:-----|:-----------|
| `int` / `float` / `string` | 基础类型 | `1001, 3.14, "Sword"` |
| `bool` | 布尔值 | `true` / `false` |
| `enum` | 枚举（需在 `__enums__.xlsx` 定义） | `EItemType.Weapon` |
| `list<T>` / `array<T>` | 列表/数组 | `1,2,3` |
| `map<K,V>` | 字典 | `key1:val1,key2:val2` |
| `bean` | 内嵌结构体 | 多列展开 |
| `datetime` | 日期时间 | `2024-01-15 10:00:00` |
| Foreign Key | 外键引用 | `##ref` 语法 |

## 多格式输出与校验

同一套 Excel 可同时输出 JSON（开发调试）、Binary（发布包，体积最小）、Lua（热更，零解析开销）、C#（编译期强类型）。

Luban 在生成阶段执行完整校验：类型检查、外键完整性、范围验证、唯一性约束——错误在**编译期暴露**，非运行时崩溃。

## 与 [[concepts/XLua热补丁|XLua 热补丁]] 配合

1. **配置表入包**（Binary）：发布时打入包体，加载最快
2. **热更新配置**（JSON）：通过 Lua 热更包下发
3. **Lua 格式输出**：直接生成 `.lua` 数据文件，Lua 侧零开销使用

```lua
local cfg = require("Config.item")
local sword = cfg[1001]
print(sword.Name, sword.Price)
```

## 常见面试要点

1. **Luban 解决了什么问题**：类型安全（编译期校验替代运行时字符串解析）、多格式输出、策划友好
2. **为什么选择 Luban 而非手写配置系统**：内置校验系统、外键/唯一性检查、多格式一键切换
3. **Binary vs JSON vs Lua 输出选择**：发布用 Binary（体积小加载快），开发用 JSON（可读），Lua 热更用 Lua 格式（零解析开销）
4. **Luban 如何与 XLua 配合实现热更**：配置表可走 Lua 热更通道，Luban 直接输出 Lua 数据文件
5. **加载回调的设计**：`Tables` 构造传入加载委托，支持任意来源（Resources/AB/网络），依赖表自动递归加载

## 参见

- [[sources/luban-config-摘要|Luban 配置工具 — 来源摘要]]
- [[concepts/XLua热补丁|XLua 热补丁]]
- [[concepts/Unity脚本架构|Unity 脚本架构]]
- [[concepts/Unity资源管理|Unity 资源管理]]
