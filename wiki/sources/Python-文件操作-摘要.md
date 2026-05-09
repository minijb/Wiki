---
title: "Python 文件操作 — pathlib/os/shutil"
type: source-summary
updated: 2026-05-07
source: "raw/cs/languages/Python-文件操作.md"
tags: [python, file, io, pathlib, 文件系统]
---

# Python 文件操作 — pathlib/os/shutil

Python 中文件和目录的增删改查操作，覆盖 `pathlib`、`os`、`shutil`、`glob`、`fnmatch` 五个标准库。

## 核心要点

- **`pathlib.Path` 是首选**：Python 3.4+ 的面向对象路径 API，用 `/` 运算符拼接路径，内置读写方法，替代 `os.path` 的大部分功能
- **分层选型**：日常路径处理用 `pathlib`；高级复制/移动/压缩用 `shutil`；底层系统调用和 `os.walk()` 遍历用 `os`
- **安全写入**：原子写入模式 — 先写临时文件，`fsync()` 刷盘，再 `os.replace()` 原子替换
- **路径安全**：用户输入路径须用 `resolve()` + `is_relative_to()` 防止目录遍历攻击

## 模块定位

| 模块 | 定位 |
|------|------|
| `pathlib` | 现代 OOP 路径操作，官方推荐首选 |
| `os` / `os.path` | 底层系统调用，兼容旧代码 |
| `shutil` | 高级复制/移动/压缩/递归删除 |
| `glob` | 通配符批量查找文件 |
| `fnmatch` | Unix shell 风格文件名匹配 |

## 相关概念

- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — 设计哲学、模块分层和安全模型
