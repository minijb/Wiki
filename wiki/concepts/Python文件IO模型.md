---
title: "Python 文件 I/O 模型"
type: concept
updated: 2026-05-10
tags: [python, file, io, pathlib, 安全]
aliases: [Python文件IO, Python路径操作]
---

# Python 文件 I/O 模型

Python 文件操作的标准库分层架构、设计哲学与安全实践。

## 库的分层架构

```
┌─────────────────────────────────┐
│  pathlib (OOP, 现代首选)         │  ← Path("/a") / "b" / "c"
├─────────────────────────────────┤
│  shutil (高级操作)               │  ← copy, move, rmtree, archive
├─────────────────────────────────┤
│  os / os.path (底层系统调用)     │  ← mkdir, remove, rename, walk
├─────────────────────────────────┤
│  open() (内置文件读写)           │  ← 所有库最终都通过 open() 读写
└─────────────────────────────────┘
```

## 设计哲学

### pathlib: 面向对象路径

`pathlib.Path` 将路径抽象为对象，方法直接挂载在路径上：

```python
# os.path 风格（函数式）
basename = os.path.basename(p)
dirname  = os.path.dirname(p)

# pathlib 风格（OOP）
basename = p.name
dirname  = p.parent
```

优势：
- `/` 运算符拼接路径，自然的可读性
- 方法链：`p.parent.parent.name`
- 内置读写：`p.read_text()` 无需额外 `open()`
- 内置 glob：`p.rglob("*.py")` 替代 `glob` 模块

### 选型原则

| 场景 | 推荐 |
|------|------|
| 日常路径处理 | `pathlib.Path` |
| 复制/移动/压缩 | `shutil` |
| 递归遍历目录树 | `os.walk()` 或 `pathlib.Path.rglob()` |
| 递归删除非空目录 | `shutil.rmtree()` |
| 批量查找文件 | `pathlib.Path.rglob()` |
| 兼容 Python < 3.4 | `os.path` |

## 操作分类

### 查询（只读）

- 存在性：`exists()`, `is_file()`, `is_dir()`
- 元数据：`stat()`, `getsize()`, `getmtime()`
- 遍历：`listdir()`, `scandir()`, `walk()`, `iterdir()`, `rglob()`
- 路径计算：`basename()`, `dirname()`, `splitext()`, `parent`, `stem`, `suffix`

### 创建与写入

- 目录：`mkdir()` / `makedirs()`，配合 `exist_ok=True`
- 文件：`open()` 的 `w`（覆盖）、`a`（追加）、`x`（独占创建）模式
- 直接写入：`pathlib.Path.write_text()` / `write_bytes()`

### 修改

- `rename()` / `replace()` — 同文件系统元数据操作
- `move()` — 跨文件系统降级为复制+删除

### 删除

- `remove()` / `unlink()` — 删除文件
- `rmdir()` — 删除空目录
- `rmtree()` — 递归删除（破坏性，不可恢复）

## 安全模型

### 原子写入

关键数据写入应使用原子模式——避免写入中断导致文件损坏：

```
1. 在同目录创建临时文件（确保 rename 是原子的）
2. 写入内容 + fsync() 刷盘
3. os.replace() 原子替换目标文件
4. 异常时清理临时文件
```

### 路径遍历防护

用户提供的路径必须验证不逃逸出基准目录：

```python
resolved = (base_dir / user_path).resolve()
if not resolved.is_relative_to(base_dir.resolve()):
    raise ValueError("非法路径")
```

## 交叉引用

- [[sources/Python-文件操作-摘要|Python 文件操作来源摘要]] — 完整 API 参考和代码示例
- [[concepts/Python子进程管理|Python 子进程管理]]
- [[concepts/CSharp文件IO|C# 文件 I/O]] — File/FileStream 分层架构与异步 I/O 对比
