---
title: Python 数据处理与杂项
updated: 2026-06-02
tags:
  - python
  - mysql
  - javascript
  - opencv
  - matplotlib
type: language
description: Python 性能检测、图像处理（PIL/OpenCV/Matplotlib）、MySQL 基础操作、JavaScript 基础语法等杂项知识汇总
---

# Python 数据处理与杂项

> [!note] 已有专题页面
> Python 的文件 I/O 和子进程管理已有独立页面：
> - [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib / shutil / os 分层架构、原子写入、路径安全
> - [[concepts/Python子进程管理|Python 子进程管理]] — subprocess.run() / Popen() 接口分层、命令注入防护、管道死锁
>
> 以下内容为上述专题之外的数据处理与工具类知识点。

---

## Python 性能检测

Python 提供了多种性能分析工具，从高层概览到底层逐行分析：

### cProfile — 标准库性能分析器

`cProfile` 是 Python 内置的确定性性能分析器（C 扩展实现，开销低），记录每个函数的调用次数、总耗时和自身耗时。

```python
import cProfile

cProfile.run("my_function()")
# 输出：按调用次数/总耗时/自身耗时排序的函数统计表

# 或从命令行运行
# python -m cProfile -s cumulative myscript.py
```

常用排序键：`cumulative`（累积耗时）、`time`（自身耗时）、`calls`（调用次数）、`ncalls`（总调用次数）。

### snakeviz — 可视化分析结果

[snakeviz](https://jiffyclub.github.io/snakeviz/) 是一个基于浏览器的 cProfile 结果可视化工具，将统计输出转换为交互式火焰图（Sunburst）和表格：

```bash
python -m cProfile -o output.prof myscript.py
snakeviz output.prof
# 浏览器打开交互式界面：鼠标悬停查看详情，点击钻取到子函数
```

### line-profiler — 逐行性能分析

`line-profiler` 定位到**具体代码行**的耗时，而非函数级别：

```python
# 安装: pip install line-profiler

# 方式一：装饰器
from line_profiler import LineProfiler

lp = LineProfiler()

@lp
def my_func():
    total = 0
    for i in range(10000):
        total += i * i      # ← 可精确知道这一行耗时多少
    return total

my_func()
lp.print_stats()
```

```bash
# 方式二：命令行（需在目标函数上加 @profile 装饰器）
kernprof -l -v myscript.py
```

输出示例：

```
Line #      Hits      Time  Per Hit   % Time  Line Contents
     5     10000     0.003      0.0      1.5      total += i * i
```

### 选型建议

| 工具 | 适用场景 |
|------|---------|
| `cProfile` | 快速识别热点函数 |
| `snakeviz` | 可视化探索调用关系 |
| `line-profiler` | 定位热点代码行 |
| `timeit` | 微基准测试（小代码片段） |

---

## Python 图像处理

Python 生态中三大图像处理库——PIL (Pillow)、OpenCV、PyTorch (torchvision)——各有不同的数据格式、维度顺序和颜色通道约定。

### PIL / Pillow

Pillow 是 Python 图像处理标准库，格式为 `(宽, 高)`：

```python
from PIL import Image

image = Image.open("path/to/image.png")
# image.size → (width, height)
# image.mode → "RGB" / "RGBA" / "L" 等

# 转换为 RGB（丢弃 alpha 通道）
image = Image.open("image.png").convert("RGB")

# 保存
image.save("output.jpg", quality=95)
```

### PyTorch Tensor 图像

torchvision 图像格式为 `(通道数, 高, 宽)`，数据类型 `float32`，值域 `[0.0, 1.0]`，颜色通道顺序 **RGB**：

```python
from torchvision import transforms
from PIL import Image

# PIL → Tensor（自动缩放到 [0,1]）
image = Image.open("image.jpg").convert("RGB")
tensor = transforms.ToTensor()(image)
# tensor.shape → (3, height, width), dtype=float32

# Tensor → PIL
pil_img = transforms.ToPILImage()(tensor)
# 或使用 ToPILImage(mode) 指定模式
```

### OpenCV (cv2)

OpenCV 图像格式为 `(高, 宽, 通道数)`，数据类型 `uint8`，底层为 `numpy.ndarray`，颜色通道顺序 **BGR**（与 PIL/RGB 相反）。

### 基本操作

```python
import cv2

# 读取
img = cv2.imread("path/to/image.jpg")
# img.shape → (height, width, 3), dtype=uint8

# 显示
cv2.imshow("Window", img)
cv2.waitKey(0)             # 等待按键，0 表示无限等待
cv2.destroyAllWindows()    # 关闭所有窗口

# 写入
cv2.imwrite("output.png", img)
```

### 图像格式转换

三种库之间的格式转换核心是**维度重排**和**颜色通道变换**：

**Tensor → OpenCV**：

```python
import numpy as np
import cv2

# tensor: (C, H, W), float32, RGB
max_val = tensor_img.max()
tensor_img = tensor_img * 255 / max_val        # 缩放到 [0,255]
cv_img = tensor_img.permute(1, 2, 0).numpy()   # (H, W, C)
cv_img = np.uint8(cv_img)                       # float32 → uint8
cv_img = cv2.cvtColor(cv_img, cv2.COLOR_RGB2BGR)  # RGB → BGR
```

**Tensor → PIL**：

```python
from torchvision import transforms

img_pil = transforms.ToPILImage()(img_tensor)       # 不缩放
img_tensor = transforms.PILToTensor()(img_pil)       # 不缩放，直接转

# ToTensor 默认将 uint8 [0,255] 缩放到 float [0,1]
img_tensor = transforms.ToTensor()(img_pil)
```

**PIL → OpenCV（统一到通道优先格式后比较）**：

```python
from PIL import Image
from torchvision import transforms
import cv2

img_pil = Image.open("image.png")
img_cv2 = cv2.imread("image.png")

# 统一为 (C, H, W) 格式进行比较
pil_numpy = transforms.PILToTensor()(img_pil).numpy()  # (C, H, W)
cv2_numpy = img_cv2.transpose(2, 0, 1)                  # (H, W, C) → (C, H, W)
```

### 格式速查表

| 库 | 维度顺序 | 数据类型 | 值域 | 颜色通道 |
|----|---------|---------|------|---------|
| PIL | `(W, H)` | — | — | RGB |
| Tensor | `(C, H, W)` | `float32` | `[0.0, 1.0]` | RGB |
| OpenCV | `(H, W, C)` | `uint8` | `[0, 255]` | BGR |

---

## Matplotlib 图像展示

`matplotlib.pyplot.imshow()` 是将数据渲染为图像的核心函数。

### 基本用法

```python
import matplotlib.pyplot as plt

plt.imshow(image)
plt.title("My Image")
plt.axis("off")   # 隐藏坐标轴
plt.show()
```

### image 参数支持的格式

- `(M, N)` — 标量数据（单通道），使用颜色映射（colormap）将值映射到颜色
- `(M, N, 3)` — RGB 值，`[0.0, 1.0]` 浮点数或 `[0, 255]` 整数
- `(M, N, 4)` — RGBA 值，同上值域范围

### 关键参数

| 参数 | 说明 |
|------|------|
| `cmap` | 颜色映射。常用：`'viridis'`（默认）、`'gray'`、`'hot'`。`plt.colormaps()` 可查看全部可用映射 |
| `aspect` | 坐标轴纵横比。`'auto'` 自动调整，`'equal'` 强制等比例 |
| `interpolation` | 插值方法。`'nearest'`（无插值）、`'bilinear'`、`'bicubic'`（更平滑但计算成本更高） |
| `vmin` / `vmax` | 限制显示的数据值范围，超出部分被截断 |

### 完整示例

```python
import matplotlib.pyplot as plt
import numpy as np

# 单通道灰度图
data = np.random.rand(10, 10)
plt.imshow(data, cmap="gray", interpolation="bilinear", vmin=0, vmax=1)
plt.colorbar()           # 显示颜色条
plt.title("Grayscale Heatmap")
plt.show()
```

---

## MySQL 基础操作

### 连接

```bash
mysql -u username -p
# 输入密码后进入交互式 shell
```

连接远程服务器需确保服务器端已开启端口并授权。

### 服务管理 (Linux)

```bash
service mysql restart    # 重启
service mysql stop       # 停止
service mysql start      # 启动
```

### 用户管理

**创建用户**：

```sql
-- 允许任意 IP 连接
CREATE USER 'alice'@'%' IDENTIFIED BY 'password123';

-- 仅允许本地连接
CREATE USER 'alice'@'localhost' IDENTIFIED BY 'password123';

FLUSH PRIVILEGES;   -- 立即生效
```

用户信息存储在 `mysql.user` 表中，可通过 `SELECT user, host FROM mysql.user;` 查询。

**删除用户**：

```sql
DROP USER 'alice'@'%';
```

### 授权

```sql
-- 授予所有数据库的所有权限（允许任意 IP）
GRANT ALL PRIVILEGES ON *.* TO 'alice'@'%' WITH GRANT OPTION;

-- 授予指定数据库的指定权限
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'alice'@'%';

-- 仅本地登录
GRANT ALL PRIVILEGES ON *.* TO 'alice'@'localhost' IDENTIFIED BY 'password123' WITH GRANT OPTION;

FLUSH PRIVILEGES;   -- 刷新权限表
```

`GRANT` 语句结构：

| 子句 | 说明 |
|------|------|
| `ALL PRIVILEGES` | 所有权限；也可指定 `SELECT`、`CREATE`、`DROP` 等 |
| `ON 数据库.表` | `*.*` 表示所有库所有表；`test.user` 表示 test 库的 user 表 |
| `TO '用户名'@'登录IP'` | `%` 表示任意 IP，`localhost` 仅本地 |
| `IDENTIFIED BY '密码'` | 可选，同时设置密码 |
| `WITH GRANT OPTION` | 允许用户将自己的权限授予他人 |

权限信息存储在 `mysql.db` 表中。

### 修改密码

```sql
USE mysql;
UPDATE user SET authentication_string='' WHERE user='root';

ALTER USER 'root'@'localhost'
    IDENTIFIED WITH mysql_native_password BY 'new_password';

FLUSH PRIVILEGES;
```

### 远程连接 (Linux)

在服务器上配置 MySQL 允许远程访问：

1. 创建允许远程登录的用户（见上文 `'user'@'%'`）
2. 编辑 MySQL 配置文件：

```bash
vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

将 `bind-address = 127.0.0.1` 改为 `bind-address = 0.0.0.0`（或注释该行），使 MySQL 监听所有网络接口。

3. 重启 MySQL 服务：

```bash
service mysql restart
```

4. 确保防火墙放行 MySQL 端口（默认 3306）。

---

## JavaScript 基础

### 脚本引入

```html
<!-- 方式一：内联脚本 -->
<script type="text/javascript">
    "use strict";
    // 代码...
</script>

<!-- 方式二：外部文件 -->
<script src="/path/to/script.js"></script>
```

> [!tip] 严格模式
> 始终在脚本或函数开头声明 `"use strict";`，启用 ES5 严格模式，禁止不安全操作并抛出更多异常。

### 变量声明

| 关键字 | 块级作用域 | 可重复声明 | 可修改 |
|--------|----------|----------|------|
| `let` | 是 | 否 | 是 |
| `const` | 是 | 否 | 否（常量） |
| `var` | 否（函数作用域） | 是 | 是 |

`var` 无块级作用域会导致意外行为，现代代码优先使用 `let` 和 `const`。

### 数据类型

**number** — 整数、浮点数、以及特殊值：

```js
let n = 42;             // 整数
let pi = 3.14;          // 浮点数
let inf = Infinity;     // 无穷大
let bad = NaN;          // 计算错误 (Not a Number)

// NaN 与任何值运算结果都是 NaN
NaN + 5;  // NaN
isNaN(NaN);  // true
```

**BigInt** — 任意精度整数，后缀 `n`：

```js
const big = 1234567890123456789012345678901234567890n;
// 不能与普通 number 混合运算
```

**string** — 三种引号：

```js
"Dobule quotes"
'Single quotes'
`Backticks — 支持 ${expression} 插值`

let name = "Alice";
console.log(`Hello, ${name}!`);      // "Hello, Alice!"
console.log(`1 + 2 = ${1 + 2}`);    // "1 + 2 = 3"
```

**boolean** — `true` / `false`

**null** — 表示"空"或"不存在"（独立类型）

**undefined** — 变量已声明但未赋值：

```js
let x;
console.log(x);  // undefined
```

**object** — 复杂数据结构集合

**symbol** — 创建唯一标识符（对象属性的唯一键）

### typeof 运算符

```js
typeof undefined   // "undefined"
typeof 0           // "number"
typeof 10n         // "bigint"
typeof true        // "boolean"
typeof "hello"     // "string"
typeof Symbol()    // "symbol"
typeof Math        // "object"
typeof null        // "object"  ← 历史遗留 bug
typeof alert       // "function"
```

> [!warning] `typeof null === "object"`
> 这是 JavaScript 自第一版延续至今的行为，已被 ECMAScript 规范承认但不会修复。判断 `null` 应使用 `value === null`。

### 浏览器交互

```js
// 模态弹窗
alert("Hello, World!");

// 带输入框的对话框（返回用户输入或 null）
let name = prompt("Your name?", "default value");

// 确认对话框（返回 true 或 false）
let ok = confirm("Are you sure?");
```

### 类型转换

**字符串转换**：

```js
let str = String(value);
// 或
let str = value + "";      // 隐式转换（较少使用）
```

**数字转换**：

```js
let num = Number("123");   // 123
let num = +"456";          // 456（一元 + 运算符）

// 自动转换（算术运算）
console.log("6" / "2");   // 3
```

**布尔转换**：

```js
Boolean(0)         // false
Boolean("")        // false
Boolean(null)      // false
Boolean(undefined) // false
Boolean(NaN)       // false
Boolean("hello")   // true
Boolean(42)        // true
// 空字符串、0、null、undefined、NaN → false；其余 → true
```

---

## 交叉引用

- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib / os / shutil 完整参考
- [[concepts/Python子进程管理|Python 子进程管理]] — subprocess 接口分层与安全模型
- [[concepts/CSharp文件IO|C# 文件 I/O]] — 跨语言文件操作对比
- [[concepts/CSharp进程管理|C# 进程管理]] — 跨语言进程管理对比
