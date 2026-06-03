---
title: "Python 数据处理与杂项 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/cs/languages/python-data-and-misc.md"
tags: [python, mysql, javascript, opencv, matplotlib]
---

# Python 数据处理与杂项 — 摘要

来源：`raw/cs/languages/python-data-and-misc.md`

## 概述

Python 文件操作（shutil/os/pathlib）、子进程管理（subprocess.run/Popen）、性能检测、图像处理（PIL / OpenCV / Matplotlib 三库格式互转）、MySQL 基础操作与远程连接、JavaScript 基础语法等杂项知识汇总。
## 要点
1. **文件操作** — `shutil.copy`/`copytree`/`move`（复制/移动）、`os.unlink`/`rmdir`/`shutil.rmtree`（删除）、`os.makedirs(exist_ok=True)`（创建文件夹）。推荐 `pathlib.Path` 进行路径管理
2. **子进程管理** — `subprocess.run()`（高层，等待完成）vs `subprocess.Popen()`（底层，灵活控制）；`shell=True` 存在命令注入风险，参数始终用列表形式；`capture_output=True` 同时捕获 stdout/stderr；`encoding` 指定文本编码
3. **管道死锁防护** — stdout/stderr 同时使用 PIPE 且输出量大时，`communicate()` 是最安全的方式——内部并发读取两个管道，手动先后读取可能导致死锁
2. **图像格式三库差异**：PIL 维度 `(W, H)` RGB，PyTorch Tensor 维度 `(C, H, W)` float32 `[0.0, 1.0]` RGB，OpenCV 维度 `(H, W, C)` uint8 `[0, 255]` BGR。互转核心操作为维度重排（`permute` / `transpose`）和颜色通道转换（`cvtColor`）
3. **Tensor → OpenCV 转换链**：缩放至 `[0, 255]` → `permute` 重排为 `(H, W, C)` → `.numpy()` → `uint8` 类型转换 → `cv2.COLOR_RGB2BGR` 通道转换
4. **Matplotlib imshow**：支持 `(M, N)` 标量 + colormap、`(M, N, 3)` RGB、`(M, N, 4)` RGBA。关键参数：`cmap`（颜色映射，默认 `'viridis'`）、`interpolation`（`'nearest'` 无插值 / `'bicubic'` 平滑）、`vmin` / `vmax`（值域截断）
5. **MySQL 用户与权限**：`CREATE USER 'user'@'host' IDENTIFIED BY 'pwd'`，host 用 `%` 表示任意 IP、`localhost` 仅本地；`GRANT ... ON db.table TO ...` 授予权限；`FLUSH PRIVILEGES` 立即生效。权限信息存储在 `mysql.user` 和 `mysql.db` 表中
6. **MySQL 远程连接三步**：创建 `'user'@'%'` 用户 → `/etc/mysql/mysql.conf.d/mysqld.cnf` 中设 `bind-address = 0.0.0.0` → 防火墙放行 3306 端口
7. **JS 变量声明**：`let` / `const` 块级作用域（推荐），`var` 函数作用域（遗留，无块级作用域）。`const` 声明常量不可重新赋值
8. **JS 七种原始类型**：`number`（含 `Infinity` / `NaN`）、`bigint`（后缀 `n`，不可与 number 混算）、`string`（反引号支持 `` `${}` `` 插值）、`boolean`、`null`、`undefined`、`symbol`。外加 `object`（引用类型）
9. **`typeof` 陷阱**：`typeof null === "object"` 是 JS 第一版延续至今的历史遗留 bug，被 ECMAScript 承认但不会修复。判断 `null` 应使用 `value === null`
10. **JS 类型转换**：falsy 值：`0`、`""`、`null`、`undefined`、`NaN` → `Boolean()` 转 `false`；其余转 `true`。数字转换用 `Number()` 或一元 `+`，字符串转换用 `String()` 或 `+ ""`
11. **JS 浏览器交互**：`alert()` 模态弹窗，`prompt(msg, default)` 返回输入或 `null`，`confirm()` 返回 `true` / `false`

## 关联 Wiki 页面

- [[concepts/Python数据处理与杂项|Python 数据处理与杂项]] — 概念页面
- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib / os / shutil 完整参考
- [[concepts/Python子进程管理|Python 子进程管理]] — subprocess 接口分层与安全模型
- [[concepts/CSharp文件IO|C# 文件 I/O]] — 跨语言文件操作对比
- [[concepts/CSharp进程管理|C# 进程管理]] — 跨语言进程管理对比