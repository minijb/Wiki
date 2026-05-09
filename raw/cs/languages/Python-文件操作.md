---
title: Python 文件操作
date: 2026-03-16
updated: 2026-05-07
tags:
  - python
  - file
  - io
  - pathlib
type: language
aliases:
  - Python文件操作
description: Python 中文件和目录的增删改查操作，涵盖 pathlib（现代 OOP）、os、shutil、glob 等标准库及安全实践
status: refined
---

# Python 文件的增删改查

Python 中文件操作涉及多个标准库，选型建议：

| 模块 | 用途 | 推荐度 |
|------|------|--------|
| `pathlib` | 面向对象路径操作，现代 Python 首选 | 首选 |
| `os` | 底层文件/目录操作 | 按需 |
| `os.path` | 路径字符串处理（函数式 API） | 兼容旧代码 |
| `shutil` | 高级文件/目录操作（复制、移动、压缩） | 高级操作 |
| `glob` | 文件名模式匹配 | 批量查找 |
| `fnmatch` | Unix shell 风格通配符匹配 | 自定义过滤 |

## 文件打开模式速查

| 模式 | 说明 |
|------|------|
| `"r"` | 只读（默认），文件必须存在 |
| `"w"` | 写入，覆盖已有内容，文件不存在则创建 |
| `"a"` | 追加，文件不存在则创建 |
| `"x"` | 独占创建，文件已存在则报错 |
| `"t"` | 文本模式（默认），自动处理编解码和换行符 |
| `"b"` | 二进制模式 |
| `"+"` | 读写模式（如 `"r+"`, `"w+"`） |

> 推荐始终显式指定 `encoding="utf-8"`，跨平台行为一致。

---

## pathlib — 现代路径操作（Python 3.4+）

`pathlib.Path` 是官方推荐的面向对象路径处理方式，替代 `os.path` 的大部分功能。

### 构造与路径拼接

```python
from pathlib import Path

Path.cwd()                  # 当前工作目录
Path.home()                 # 用户主目录
Path("dir/file.txt")        # 字符串构造

# 路径拼接使用 / 运算符（最显著的语法优势）
new_path = Path("root") / "dir" / "file.txt"
```

### 路径属性

```python
p = Path("/data/project/main.py")

p.parent          # Path("/data/project")   — 父目录
p.parents[0]      # 同上
p.name            # "main.py"               — 文件名（含扩展名）
p.stem            # "main"                  — 文件名（不含扩展名）
p.suffix          # ".py"                   — 扩展名
p.suffixes        # [".py"]                 — 多个扩展名列表
p.anchor          # "/"                     — 根锚点
p.parts           # ("/", "data", "project", "main.py")
p.as_posix()      # "/data/project/main.py" — 强制 Unix 风格分隔符
```

### 路径判断

```python
p.exists()        # 路径是否存在
p.is_file()       # 是否为文件
p.is_dir()        # 是否为目录
p.is_symlink()    # 是否为符号链接
p.is_absolute()   # 是否为绝对路径
p.is_relative_to(Path("/data"))  # 是否在指定目录下
```

### 目录操作

```python
p.mkdir(parents=True, exist_ok=True)  # 递归创建目录
p.rmdir()                             # 删除空目录
p.unlink()                            # 删除文件（miss_ok=True 时不存在也不报错）
p.rename(new_path)                    # 重命名/移动
p.replace(new_path)                   # 重命名，目标存在则覆盖
```

### 遍历与匹配

```python
# 遍历目录（生成器，内存友好）
for entry in p.iterdir():
    print(entry.name)

# 模式匹配
list(p.glob("*.py"))            # 当前目录下 .py 文件
list(p.rglob("**/*.py"))        # 递归匹配所有子目录
```

### 直接读写（无需 open()）

```python
content = p.read_text(encoding="utf-8")           # 读文本
p.write_text("hello world", encoding="utf-8")     # 写文本
data = p.read_bytes()                             # 读二进制
p.write_bytes(b"hello")                           # 写二进制
```

### pathlib vs os.path 对照

| 操作 | os/os.path | pathlib |
|------|-----------|---------|
| 路径拼接 | `os.path.join("a", "b")` | `Path("a") / "b"` |
| 文件名 | `os.path.basename(p)` | `p.name` |
| 无扩展名 | `os.path.splitext(p)[0]` | `p.stem` |
| 扩展名 | `os.path.splitext(p)[1]` | `p.suffix` |
| 父目录 | `os.path.dirname(p)` | `p.parent` |
| 是否存在 | `os.path.exists(p)` | `p.exists()` |
| 创建目录 | `os.makedirs(p, exist_ok=True)` | `p.mkdir(parents=True, exist_ok=True)` |
| 读全部文本 | `open(p).read()` | `p.read_text()` |
| 删除文件 | `os.remove(p)` | `p.unlink()` |

---

## os — 底层系统操作

### 目录操作

```python
import os

os.getcwd()                     # 获取当前工作目录
os.chdir(path)                  # 切换工作目录
os.mkdir(path)                  # 创建单层目录，父目录必须存在
os.makedirs(path, exist_ok=True) # 递归创建目录
os.listdir(path)                # 列出目录下所有条目，返回 list
os.scandir(path)                # 返回迭代器，每个条目含 stat 信息，性能更好
os.rmdir(path)                  # 删除空目录（目录非空时报错）
os.removedirs(path)             # 递归删除空目录
```

### 递归遍历：os.walk()

```python
for root, dirs, files in os.walk("/path/to/dir"):
    for name in files:
        print(os.path.join(root, name))  # 每个文件的完整路径
    for name in dirs:
        print(os.path.join(root, name))  # 每个子目录的完整路径
```

> `os.walk()` 默认自顶向下。`dirs[:]` 就地修改可控制遍历范围。

### 文件操作

```python
os.remove(path)      # 删除文件（文件不存在时报错）
os.unlink(path)      # 删除文件，Unix 下与 remove 相同
os.rename(src, dst)  # 重命名文件或目录（仅限同一文件系统）
os.stat(path)        # 获取文件/目录元信息（大小、权限、时间戳等）
os.access(path, os.R_OK | os.W_OK)  # 检查读写权限
```

### 路径处理：os.path

```python
import os.path

os.path.exists(path)       # 路径是否存在（文件或目录均可）
os.path.isfile(path)       # 是否为文件
os.path.isdir(path)        # 是否为目录
os.path.isabs(path)        # 是否为绝对路径
os.path.getsize(path)      # 文件大小（字节）
os.path.getmtime(path)     # 最后修改时间（Unix 时间戳）

os.path.join("a", "b")     # 跨平台路径拼接
os.path.split(path)        # → (目录, 文件名)
os.path.splitext(path)     # → (路径, 扩展名)
os.path.basename(path)     # 获取文件名
os.path.dirname(path)      # 获取目录路径
os.path.abspath(path)      # 相对路径 → 绝对路径
os.path.relpath(path, start)  # 计算相对路径
os.path.normpath(path)     # 规范化路径（消除 .. 和 .）
```

> **注意**：`os.path.join("a/", "/b")` 会丢弃前面路径只返回 `"/b"`，因为 `/b` 是绝对路径。

---

## shutil — 高级文件操作

### 复制

```python
import shutil

shutil.copy(src, dst)       # 复制文件（dst 可以是目录或文件名）
shutil.copy2(src, dst)      # 复制文件，同时保留元数据（atime/mtime）
shutil.copytree(src, dst)   # 递归复制整个目录树（dst 必须不存在）
shutil.copytree(src, dst, dirs_exist_ok=True)  # Python 3.8+: 目标存在也允许
```

### 移动与删除

```python
shutil.move(src, dst)       # 移动文件/目录，跨文件系统时自动降级为复制+删除
shutil.rmtree(path)         # 递归删除非空目录
shutil.rmtree(path, ignore_errors=True)  # 忽略删除过程中的错误
```

> **警告**：`shutil.rmtree()` 是破坏性操作，删除的内容不可恢复。执行前务必确认路径正确。

### 压缩与解压

```python
shutil.make_archive("backup", "zip", root_dir="folder")      # 压缩 folder → backup.zip
shutil.make_archive("backup", "gztar", root_dir="folder")    # tar.gz 格式
shutil.unpack_archive("backup.zip", "extract_dir")           # 解压
shutil.get_archive_formats()  # 查看支持的压缩格式
shutil.get_unpack_formats()   # 查看支持的解压格式
```

### 磁盘信息

```python
usage = shutil.disk_usage("/")
# usage.total   — 总容量
# usage.used    — 已用
# usage.free    — 可用
```

---

## glob / fnmatch — 模式匹配

```python
import glob

glob.glob("*.py")                        # 当前目录下 .py 文件
glob.glob("**/*.py", recursive=True)     # 递归匹配所有子目录
list(glob.iglob("**/*.py", recursive=True))  # 生成器版本，节省内存

# pathlib 内置 glob 更推荐：
from pathlib import Path
list(Path(".").rglob("*.py"))
```

```python
import fnmatch

# 自定义文件名过滤（不依赖文件系统）
fnmatch.fnmatch("file.txt", "*.txt")        # True
fnmatch.filter(["a.py", "b.txt"], "*.py")   # ["a.py"]
```

---

## 路径遍历安全

当允许用户指定路径时，必须防止路径遍历攻击（如 `../../etc/passwd`）：

```python
from pathlib import Path

def safe_resolve(base_dir: Path, user_path: str) -> Path:
    """将用户路径解析到 base_dir 内，防止逃逸"""
    resolved = (base_dir / user_path).resolve()
    if not resolved.is_relative_to(base_dir.resolve()):
        raise ValueError(f"非法路径: {user_path}")
    return resolved

# 用法
try:
    safe = safe_resolve(Path("uploads"), "../../../etc/passwd")
except ValueError:
    print("路径越界被拦截")
```

> `is_relative_to()` 需要 Python 3.9+。更早版本可用 `str(resolved).startswith(str(base_dir.resolve()))`。

---

## 原子写入模式

```python
import os
import tempfile

def safe_write(filepath, content):
    """先写临时文件，成功后再原子替换，避免写入中断导致文件损坏。"""
    dirname = os.path.dirname(filepath) or "."
    with tempfile.NamedTemporaryFile(
        dir=dirname, delete=False, mode="w", encoding="utf-8"
    ) as tmp:
        tmp.write(content)
        tmp.flush()
        os.fsync(tmp.fileno())  # 确保刷入磁盘
    os.replace(tmp.name, filepath)  # 原子替换（Windows 也支持）
```
