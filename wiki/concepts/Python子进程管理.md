---
title: "Python 子进程管理"
type: concept
updated: 2026-05-10
tags: [python, subprocess, 进程, 安全]
aliases: [subprocess, Python进程]
---

# Python 子进程管理

从 `subprocess` 模块提炼的子进程管理设计哲学、选择框架与安全模型。

## 接口分层设计

`subprocess` 提供两个抽象层级：

| 层级 | API | 特点 |
|------|-----|------|
| 高层 | `subprocess.run()` | 同步阻塞，一行完成执行+等待+收集结果。**覆盖绝大多数场景。** |
| 底层 | `subprocess.Popen()` | 异步非阻塞，手动控制进程生命周期。用于流式 I/O、后台进程、管道链接。 |

选择原则：**默认用 `run()`，只在需要流式输出或双向通信时降到 `Popen()`。**

## 执行模型

```
run(args)                           Popen(args)
  │                                    │
  ├─ 创建子进程                        ├─ 创建子进程（立即返回）
  ├─ 等待子进程结束（阻塞）            ├─ 主进程继续执行
  ├─ 收集 stdout/stderr                ├─ 手动 poll() / wait()
  └─ 返回 CompletedProcess             └─ communicate() 收集结果
```

## 安全模型

### 命令注入防护

```python
# ❌ shell=True + 用户输入 → 命令注入
subprocess.run(f"rm {user_path}", shell=True)

# ✅ list 参数 → 参数独立传递，不被 shell 解析
subprocess.run(["rm", user_path])
```

核心原则：**永远不要将用户输入拼接到 shell 命令字符串中。** 传递 `list` 参数确保每个元素作为独立参数传递，不经过 shell 解释。

### 辅助工具

- `shlex.split(cmd_str)` — 安全地将 shell 命令字符串解析为 list（不经过 shell）
- `shlex.quote(s)` — 转义字符串，使其可安全嵌入 shell 命令（不得已时的最后手段）

### 环境隔离

子进程默认继承父进程环境变量。对外部命令应传递显式的 `env` 字典，避免敏感变量泄漏：

```python
subprocess.run(["cmd"], env=os.environ.copy())
```

## 管道与死锁

当 `stdout=PIPE` 或 `stderr=PIPE` 时，管道有固定缓冲区（通常 64KB）。子进程输出超过缓冲区容量时阻塞等待读取，而父进程可能也在等待子进程结束 — 形成死锁。

**解决方案：**
- `run()` 场景：使用 `capture_output=True`，内部自动安全读取
- `Popen()` 场景：使用 `communicate()`，内部通过线程异步读取
- 流式场景：逐行读取 `process.stdout` 后调用 `process.wait()`

## 超时与容错

每个子进程调用都应有超时和错误处理：

```
FileNotFoundError  → 命令不存在
TimeoutExpired     → 超时（保留已捕获的部分输出）
CalledProcessError → 返回码非零（check=True 时）
```

## 交叉引用

- [[sources/Python-运行命令行-摘要|Python 运行命令行来源摘要]] — 完整 API 参考和代码示例
- [[concepts/Python文件IO模型|Python 文件 I/O 模型]]
- [[concepts/CSharp进程管理|C# 进程管理]] — Process 类与 subprocess 设计哲学对比
