---
title: Python 运行命令行
date: 2026-03-16
updated: 2026-05-07
tags:
  - python
  - cli
  - subprocess
type: language
aliases:
  - Python命令行
description: Python 中通过 subprocess 模块执行外部命令、管理子进程的方法与最佳实践，涵盖安全防护、管道链接、异常处理
status: refined
---

# Python 运行命令行

Python 中执行外部命令的首选模块是 `subprocess`，它取代了老式的 `os.system()` 和 `os.spawn*()`。

## 核心概念

`subprocess` 模块允许你创建子进程、连接其输入/输出/错误管道、获取返回码。

两个核心接口：

| 接口 | 说明 |
|------|------|
| `subprocess.run()` | 高层封装，运行命令并等待完成，返回 `CompletedProcess`。**适合大多数场景。** |
| `subprocess.Popen()` | 底层类，灵活执行命令，支持实时交互。**需要流式输出或双向通信时使用。** |

## subprocess.run() 参数详解

```python
subprocess.run(args, *, stdin=None, input=None, stdout=None, stderr=None,
               capture_output=False, shell=False, cwd=None, timeout=None,
               check=False, encoding=None, errors=None, text=None, env=None,
               universal_newlines=None)
```

### 必选参数

- **args** — 命令和参数，推荐传 `list`（如 `["ls", "-al"]`），避免 shell 注入。若配合 `shell=True` 可传 `str`。

### 输入输出控制

- **stdin / stdout / stderr** — 重定向标准流。可取值：
  - `subprocess.PIPE` — 管道捕获（配合 `result.stdout` 读取）
  - `subprocess.DEVNULL` — 丢弃输出
  - 已存在的文件描述符或文件对象
  - `None` — 继承父进程（默认）
- **capture_output** — 设为 `True` 时自动将 stdout 和 stderr 设为 `PIPE`，等效于 `stdout=PIPE, stderr=PIPE`。
- **input** — 传给子进程 stdin 的字节串，需要 `encoding` 或 `text=True` 时可传字符串。
- **text** — 设为 `True` 时以文本模式处理 stdin/stdout/stderr（自动编解码）。等价于 `universal_newlines=True`。
- **encoding / errors** — 指定文本编解码的字符集和错误处理策略（如 `encoding='utf-8', errors='ignore'`）。

### 执行控制

- **shell** — 设为 `True` 时通过系统 shell 执行命令。**有安全风险**，切勿将用户输入拼接到 shell 命令中。仅在需要 shell 内置命令（如 `dir`、`echo`）或通配符展开时才使用。
- **cwd** — 指定子进程的工作目录。
- **env** — 指定子进程的环境变量字典。默认继承 `os.environ`。
- **timeout** — 超时秒数，超时后终止子进程并抛出 `TimeoutExpired`。

### 结果处理

- **check** — 设为 `True` 时，若返回码非零则抛出 `CalledProcessError`。适合命令不允许失败的场景。

### 返回值 `CompletedProcess`

- `result.returncode` — 进程退出码（0 表示成功）
- `result.stdout` — 捕获的标准输出（需设置 `capture_output=True`）
- `result.stderr` — 捕获的标准错误
- `result.args` — 执行的命令
- `result.check_returncode()` — 手动检查返回码，非零则抛 `CalledProcessError`

## subprocess.Popen() 详解

`Popen` 是 `run()` 的底层实现，构造函数参数与 `run()` 基本一致，但**不等待子进程完成** — 构造函数立即返回。

### 核心方法

| 方法 | 说明 |
|------|------|
| `poll()` | 非阻塞检查进程是否结束。返回 `None` 表示仍在运行，否则返回 `returncode` |
| `wait(timeout)` | 阻塞等待进程完成，可设超时 |
| `communicate(input, timeout)` | 安全地读写管道，避免死锁。返回 `(stdout, stderr)` |
| `terminate()` | 发送 SIGTERM（优雅终止） |
| `kill()` | 发送 SIGKILL（强制杀死） |
| `send_signal(sig)` | 发送指定信号 |

### 流式读取输出

```python
process = subprocess.Popen(
    ["some-long-running-command"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True
)
for line in process.stdout:
    print(line, end="")
process.wait()
```

### 管道死锁与 communicate()

当使用 `stdout=PIPE` 或 `stderr=PIPE` 时，管道缓冲区有大小限制。如果子进程输出超过缓冲区容量而你未及时读取，进程会**死锁**。

`communicate()` 通过在内部线程中读取管道来避免此问题：

```python
process = subprocess.Popen(
    ["generate-huge-output"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)
stdout, stderr = process.communicate(timeout=30)  # 安全读写，无死锁风险
```

### Popen vs run 选择指南

| 场景 | 推荐 |
|------|------|
| 执行命令，等结果 | `run()` |
| 超时控制 | `run(timeout=...)` |
| 检查返回码 | `run(check=True)` |
| 流式读取输出 | `Popen()` |
| 同时读写 stdin/stdout | `Popen()` + `communicate()` |
| 后台常驻进程 | `Popen()` + `poll()` |

## 进程管道链接

将多个进程的输入输出串联（替代 shell 管道）：

```python
# 等效于: ls -l | grep txt
p1 = subprocess.Popen(["ls", "-l"], stdout=subprocess.PIPE, text=True)
p2 = subprocess.Popen(["grep", "txt"], stdin=p1.stdout, stdout=subprocess.PIPE, text=True)
p1.stdout.close()  # 关键：让 p1 在 p2 退出时收到 SIGPIPE
output, _ = p2.communicate()
```

> 优先用 Python 原生代码替代管道链（如 `glob` + `str.find` 替代 `ls | grep`），更可维护。

## 安全防护

### 命令注入风险

```python
# ❌ 危险 — 用户输入被 shell 解析
user_input = "file.txt; rm -rf /"
subprocess.run(f"cat {user_input}", shell=True)

# ✅ 安全 — 参数作为独立元素传递
subprocess.run(["cat", user_input])
```

### 安全解析 shell 命令字符串

```python
import shlex

# shlex.split() — 按 shell 语法拆分命令字符串（不经过 shell 执行）
cmd = "git commit -m 'hello world'"
subprocess.run(shlex.split(cmd))  # 安全

# shlex.quote() — 转义单个参数（不得已用 shell=True 时）
import shlex
user_file = "file with spaces.txt"
subprocess.run(f"cat {shlex.quote(user_file)}", shell=True)
```

### 环境变量隔离

```python
import os

# 从当前环境复制后修改（推荐）
safe_env = os.environ.copy()
safe_env["MY_VAR"] = "value"

# 最简受限环境
restricted_env = {"PATH": "/usr/bin:/bin", "HOME": "/tmp"}

subprocess.run(["cmd"], env=safe_env)
```

## 完整异常处理

```python
try:
    result = subprocess.run(
        ["some_command"],
        capture_output=True, text=True, check=True, timeout=30
    )
except FileNotFoundError:
    print("命令未找到，检查 PATH 或命令名拼写")
except subprocess.CalledProcessError as e:
    print(f"命令返回非零退出码 {e.returncode}: {e.stderr}")
except subprocess.TimeoutExpired as e:
    print(f"命令超时（{e.timeout}s），已自动终止")
    # e.stdout 和 e.stderr 保留了超时前已捕获的部分输出
except OSError as e:
    print(f"系统级错误: {e}")
```

## 常见陷阱速查

| 陷阱 | 修复 |
|------|------|
| `capture_output=True` 但忘记 `text=True`，stdout 是 bytes | 加 `text=True` |
| `shell=True` + 用户输入 | 改用 list 参数 + `shlex.split()` |
| `stdout=PIPE` 未及时读取 → 死锁 | 用 `communicate()` 替代手动 `read()` |
| 未设 `timeout` → 命令可能永久挂起 | 始终设合理的 `timeout` |
| 未检查 `returncode` → 静默失败 | 设 `check=True` 或手动检查 |
| `Popen` 对象未关闭 → 资源泄漏 | 用 `with` 语句或显式 `wait()` + 关闭管道 |

## 跨平台注意事项

```python
import os

# 检查操作系统
if os.name == "nt":        # Windows
    cmd = ["dir"]
else:                      # Unix/Linux/macOS
    cmd = ["ls", "-l"]

# 文件描述符 vs PIPE 在不同平台的差异
# Windows: PIPE 通过 CreatePipe 实现，不同于 Unix pipe
# 跨平台时优先用 communicate() 而非手动 read()
```

## 实际示例

### 基本用法

```python
import subprocess

result = subprocess.run(
    ["git", "status"],
    capture_output=True,
    text=True,
    encoding='utf-8'
)
print(result.stdout)
if result.returncode != 0:
    print(f"错误: {result.stderr}")
```

### 带超时和错误检查

```python
try:
    result = subprocess.run(
        ["python", "slow_script.py"],
        capture_output=True, text=True,
        timeout=30, check=True
    )
except subprocess.TimeoutExpired:
    print("命令执行超时")
except subprocess.CalledProcessError as e:
    print(f"命令返回非零退出码: {e.returncode}")
```

### 带输入数据

```python
result = subprocess.run(
    ["python", "-c", "print(input().upper())"],
    input="hello world",
    capture_output=True, text=True
)
print(result.stdout)  # HELLO WORLD
```

### 后台进程监控

```python
import time

process = subprocess.Popen(
    ["long-running-server"],
    stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
)

while process.poll() is None:
    print("等待服务就绪...")
    time.sleep(1)

if process.returncode != 0:
    print(f"进程异常退出: {process.returncode}")
```
