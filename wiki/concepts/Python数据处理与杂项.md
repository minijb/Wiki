---
title: "Python 数据处理与杂项"
type: concept
updated: 2026-06-02
tags: [python, mysql, javascript, opencv, matplotlib, pil]
aliases: [Python图像处理, Python性能检测, MySQL基础]
---

# Python 数据处理与杂项

Python 生态中与数据打交道的工具链：性能检测、图像处理库互转、以及 MySQL/JavaScript 等跨语言基础。

> [!note] 文件操作与子进程管理
> Python 文件操作（`shutil` / `os` / `pathlib`）和子进程管理（`subprocess.run` / `Popen`）已拆分到专项页面 [[Python文件IO模型|Python 文件 I/O 模型]] 和 [[Python子进程管理|Python 子进程管理]]。

## Python 性能检测

三个层级的性能分析工具：

| 工具 | 层级 | 场景 |
|------|------|------|
| `cProfile` | 函数级 | 标准库内置，快速定位热点函数 |
| `snakeviz` | 可视化 | 浏览器交互式火焰图，钻取调用链 |
| `line-profiler` | 逐行级 | 精确定位慢代码行 |

### cProfile

```python
import cProfile

cProfile.run("my_function()")
# 输出按 ncalls / tottime / cumtime 排序的函数统计表
```

命令行：`python -m cProfile -s cumulative myscript.py`

### snakeviz

```bash
python -m cProfile -o output.prof myscript.py
snakeviz output.prof
# → 浏览器打开交互式 Sunburst 图
```

### line-profiler

```python
from line_profiler import LineProfiler
lp = LineProfiler()

@lp
def compute():
    for i in range(10000):
        x = i * i    # ← 精确到这行的耗时和调用次数

compute()
lp.print_stats()
```

## 图像库格式互转

Python 图像处理三大库**维度顺序、数据类型、颜色通道**各不相同。

### 格式速查

| 库 | 维度 | dtype | 值域 | 通道顺序 |
|----|------|-------|------|---------|
| PIL | `(W, H)` | — | — | RGB |
| PyTorch | `(C, H, W)` | `float32` | `[0, 1]` | RGB |
| OpenCV | `(H, W, C)` | `uint8` | `[0, 255]` | BGR |

### PIL (Pillow)

```python
from PIL import Image

img = Image.open("img.png")
img = img.convert("RGB")    # 统一通道
img.save("out.jpg", quality=95)
```

### PyTorch Tensor

```python
from torchvision import transforms

# PIL → Tensor
tensor = transforms.ToTensor()(pil_img)
# tensor.shape → (3, H, W), float32, [0,1]

# Tensor → PIL
pil = transforms.ToPILImage()(tensor)
```

### OpenCV

```python
import cv2

img = cv2.imread("img.jpg")   # (H, W, 3), uint8, BGR
cv2.imshow("win", img)
cv2.waitKey(0)
cv2.destroyAllWindows()
cv2.imwrite("out.png", img)
```

### 转换关键操作

**Tensor → OpenCV**：`permute(1,2,0)` 重排维度 → `numpy()` 转 ndarray → `np.uint8()` 强转类型 → `cvtColor(..., COLOR_RGB2BGR)` 换通道。

**PIL → OpenCV**：`transforms.PILToTensor()` 统一到 `(C, H, W)` 格式再比较。

## Matplotlib 图像展示

`plt.imshow()` 支持的输入格式：

- `(M, N)` 单通道 → 自动颜色映射
- `(M, N, 3)` RGB，值域 `[0,1]` 或 `[0,255]`
- `(M, N, 4)` RGBA

关键参数：

| 参数 | 说明 |
|------|------|
| `cmap` | 颜色映射：`'viridis'`（默认）、`'gray'`、`'hot'` |
| `aspect` | 纵横比：`'auto'` / `'equal'` |
| `interpolation` | 插值：`'nearest'` / `'bilinear'` / `'bicubic'` |
| `vmin` / `vmax` | 截断显示范围 |

```python
import matplotlib.pyplot as plt

plt.imshow(data, cmap="gray", interpolation="bilinear")
plt.colorbar()
plt.title("Heatmap")
plt.show()
```

## MySQL 基础

### 连接与服务管理

```bash
mysql -u root -p
service mysql restart|stop|start
```

### 用户管理

```sql
-- 创建
CREATE USER 'alice'@'%' IDENTIFIED BY 'pass123';
CREATE USER 'alice'@'localhost' IDENTIFIED BY 'pass123';

-- 授权
GRANT ALL PRIVILEGES ON *.* TO 'alice'@'%' WITH GRANT OPTION;
GRANT SELECT, INSERT ON mydb.* TO 'alice'@'%';

-- 删除
DROP USER 'alice'@'%';

FLUSH PRIVILEGES;
```

`GRANT` 语法拆解：

| 子句 | 含义 |
|------|------|
| `ALL PRIVILEGES` 或 `SELECT, INSERT, …` | 权限列表 |
| `ON db.table` | `*.*` = 所有库所有表 |
| `TO 'user'@'host'` | `%` = 任意 IP，`localhost` = 仅本地 |
| `WITH GRANT OPTION` | 允许转授权限 |

### 修改密码

```sql
ALTER USER 'root'@'localhost'
    IDENTIFIED WITH mysql_native_password BY 'newpass';
```

### 远程连接 (Linux)

1. 创建 `'user'@'%'` 用户并授权
2. 编辑 `/etc/mysql/mysql.conf.d/mysqld.cnf`，将 `bind-address` 设为 `0.0.0.0`
3. `service mysql restart`
4. 防火墙放行 3306 端口

## JavaScript 基础

### 变量声明

| 关键字 | 块作用域 | 可重复声明 | 可修改 |
|--------|---------|----------|------|
| `let` | 是 | 否 | 是 |
| `const` | 是 | 否 | 否 |
| `var` | 否 | 是 | 是 |

现代代码优先 `let` / `const`。始终声明 `"use strict";`

### 数据类型

- **number** — 整数、浮点、`Infinity`、`NaN`（NaN 任何运算都得 NaN）
- **BigInt** — 后缀 `n`：`1234567890n`
- **string** — `"..."`、`'...'`、反引号 `` `...` `` 支持 `${expr}` 插值
- **boolean** — `true` / `false`
- **null** — 空值（独立类型）
- **undefined** — 已声明未赋值
- **object** — 复杂数据结构
- **symbol** — 唯一标识符

### typeof

```js
typeof 0           // "number"
typeof "hello"     // "string"
typeof null        // "object"   ← 历史遗留
typeof alert       // "function"
```

> [!warning] `typeof null === "object"`
> 自 JS 第一版延续至今的 bug，判断 null 应使用 `value === null`。

### 浏览器交互

```js
alert("消息");
let name = prompt("提示", "默认值");   // 返回输入或 null
let ok = confirm("确认?");            // 返回 true/false
```

### 类型转换

```js
String(123)      // "123"
Number("456")    // 456
Boolean("")      // false — 空字符串
Boolean(0)       // false
Boolean("hi")    // true  — 非空字符串
```

Falsy 值：`""`、`0`、`null`、`undefined`、`NaN`。其余均为 truthy。

## 交叉引用

- [[sources/python-data-misc-摘要|Python 数据处理 — 来源摘要]]
- [[Python文件IO模型|Python 文件 I/O 模型]] — pathlib / os / shutil 完整 API
- [[Python子进程管理|Python 子进程管理]] — subprocess 安全模型
- [[CSharp文件IO|C# 文件 I/O]] — 跨语言对比
- [[CSharp进程管理|C# 进程管理]] — 跨语言对比
- [[Unity编辑器特性速查|Unity 编辑器特性速查]]
- [[Lua核心特性|Lua 核心特性]]