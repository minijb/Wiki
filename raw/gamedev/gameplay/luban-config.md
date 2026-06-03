---
title: "Luban 配置表系统"
type: source
updated: 2026-06-02
tags:
  - unity
  - luban
  - config
  - data-table
  - tooling
---

# Luban 配置表系统

Luban 是 focus-creative-games 开源的一款强大的游戏配置表解决方案，支持 Excel → 多格式（JSON、Binary、Lua、C# 等）的转换管线，内置完整的数据校验、本地化和分支合并能力。

> [!note] 核心价值
> Luban 解决了游戏开发中配置表的三大痛点：**数据类型安全**（编译期类型检查替代运行时字符串解析）、**多格式输出**（同一套 Excel 定义可输出 JSON 用于开发、Binary 用于发布、Lua 用于热更）、**策划友好**（Excel 编辑 + 实时错误提示）。

## 安装与集成

### Unity 集成

1. **安装 luban_unity 包**：

```bash
# 通过 git URL 安装（Unity Package Manager）
https://github.com/focus-creative-games/luban_unity.git
```

2. **下载 Luban 工具链**：从 [luban_example](https://github.com/focus-creative-games/luban_example) 克隆 Demo 项目，复制 `Config/` 和 `gen_config.bat` 到自己的项目。

3. **配置生成脚本**：编辑 `gen_config.bat`（Windows）或 `gen_config.sh`（macOS/Linux），设置：
   - Excel 源文件目录
   - 输出目录（JSON/Binary 等）
   - 代码生成目录（C# 类型定义）

### 基本目录结构

```
Assets/
├─ Editor/
│  └─ Luban/
│     └─ gen_config.bat    // 配置生成脚本
├─ ResExcel/                // Excel 源文件
│  ├─ item.xlsx
│  └─ hero.xlsx
└─ Generated/               // 生成的 C# 类型
   └─ Tables/
      └─ item.cs
```

## 运行时加载

Luban 按需加载配置表——通过回调函数由开发者控制加载逻辑（Resources、AssetBundle、网络下载等）：

```csharp
using Newtonsoft.Json.Linq;

// 创建 Tables 对象，传入加载回调
Tables tables = new Tables(LoadTable);

// 加载回调：根据表名返回 JSON 数据
private JArray LoadTable(string tableName)
{
    TextAsset textAsset = Resources.Load<TextAsset>($"ResExcel/JsonData/{tableName}");
    return JArray.Parse(textAsset.text);
}

// 使用配置数据
var itemCfg = tables.TbItem.Get(1001);
Debug.Log($"Item Name: {itemCfg.Name}, Price: {itemCfg.Price}");
```

> [!warning] 一次加载多个表
> `Tables` 构造函数会触发所有关联表的加载。若某个表引用了其他表的数据，Luban 会自动解析依赖并递归加载。确保回调能处理所有表名。

## 数据类型系统

Luban 支持丰富的配置数据类型：

| 类型 | 说明 | Excel 示例 |
|:-----|:-----|:-----------|
| `int` / `float` / `string` | 基础类型 | `1001, 3.14, "Sword"` |
| `bool` | 布尔值 | `true` / `false` |
| `enum` | 枚举（需在 `__enums__.xlsx` 中定义） | `EItemType.Weapon` |
| `list<T>` / `array<T>` | 列表/数组 | `1,2,3` 或 `1;2;3` |
| `map<K,V>` | 字典 | `key1:val1,key2:val2` |
| `bean`（内嵌结构体） | 复合数据 | 多列展开 |
| `datetime` | 日期时间 | `2024-01-15 10:00:00` |
| `Foreign Key` | 外键引用 | `##ref` 语法指向其他表 |

## 输出格式

Luban 支持多种输出格式，同一套 Excel 定义可同时输出为：

| 格式 | 适用场景 | 特点 |
|:-----|:---------|:-----|
| **JSON** | 开发调试、Lua 侧热更 | 可读性强，体积大 |
| **Binary** | 发布包 | 体积最小（约 JSON 的 1/5~1/10），加载快 |
| **Lua** | Lua 热更 | 直接生成 Lua table，零解析开销 |
| **C#** | 编译期类型 | 生成强类型访问代码 |

```bash
# gen_config.bat 中配置多格式输出
luban.exe \
  -t all \                          # 输出所有格式
  -c cs-simple-json \               # C# JSON 读取器
  -c cs-bin \                       # C# Binary 读取器
  -d json \                         # JSON 数据
  -d bin \                          # Binary 数据
  --conf luban.conf
```

## 数据校验

Luban 在生成阶段执行完整的数据校验，错误**在编译期暴露**，而非运行时崩溃：

- **类型检查**：整数列误填字符串 → 生成时报错
- **外键完整性**：引用了不存在的条目 → 生成时报错
- **范围验证**：超出定义的数值范围 → 生成时报错
- **唯一性约束**：主键重复 → 生成时报错
- **自定义验证规则**：通过 `validator` 字段定义

## 与 `[[XLua热补丁]]` 配合

Luban + XLua 是 Unity 热更新场景的常见组合：

1. **配置表入包**（Binary 格式）：发布时打入包体，加载最快
2. **热更新配置**（JSON 格式）：通过 Lua 热更包下发，Lua 侧直接解析 JSON
3. **Lua 格式输出**：Luban 直接生成 `.lua` 数据文件，Lua 侧零开销使用

```lua
-- Lua 侧使用 Luban 生成的配置表
local cfg = require("Config.item")
local sword = cfg[1001]
print(sword.Name, sword.Price)
```

## 参考资料

- [Luban 官方文档](https://luban.doc.default.ink/)
- [Luban GitHub](https://github.com/focus-creative-games/luban)
- [luban_unity 包](https://github.com/focus-creative-games/luban_unity)
- [Luban + Unity 使用指南](https://blog.csdn.net/SmillCool/article/details/113751711)
