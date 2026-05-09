---
title: "Python 运行命令行 — subprocess 模块"
type: source-summary
updated: 2026-05-07
source: "raw/cs/languages/Python-运行命令行.md"
tags: [python, cli, subprocess, 进程管理]
---

# Python 运行命令行 — subprocess 模块

Python 中通过 `subprocess` 模块执行外部命令与子进程管理的方法和最佳实践。

## 核心要点

- `subprocess.run()` 是高层接口，适合 90% 场景；`subprocess.Popen()` 是底层接口，用于流式输出和双向通信
- **安全性**：始终传 `list` 类型的 args 而非 `str`，避免 `shell=True` 带来的命令注入风险；必须用 shell 时用 `shlex.quote()` 转义
- 管道操作中使用 `communicate()` 避免死锁（PIPE 缓冲区满导致进程挂起）
- 始终设置 `timeout` 和 `check=True`，防止子进程永久挂起或静默失败

## 关键 API

| API | 用途 |
|-----|------|
| `subprocess.run()` | 执行命令并等待完成 |
| `subprocess.Popen()` | 灵活控制子进程生命周期 |
| `Popen.communicate()` | 安全读写管道，避免死锁 |
| `Popen.poll()` | 非阻塞检查进程状态 |
| `shlex.split()` | 安全解析 shell 命令字符串 |
| `shlex.quote()` | 转义单个参数 |

## 相关概念

- [[concepts/Python子进程管理|Python 子进程管理]] — 设计哲学、安全模型和选择框架
