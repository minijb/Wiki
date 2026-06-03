---
title: "Shell 脚本编程"
type: concept
updated: 2026-06-02
tags: [shell, bash, linux, scripting, cli]
aliases: [Shell编程, Bash脚本, Linux命令行]
---

# Shell 脚本编程

Shell 是 Linux/Unix 系统的命令行解释器，也是强大的脚本编程环境。掌握 Shell 编程是自动化运维、系统管理和开发工具链的基础。

## 基础命令与过滤器

Shell 提供了丰富的文本处理工具链：

- **`ls`**：文件列表，支持通配符（`?`、`*`、`[]`、`[!]`）
- **`grep`**：按行搜索，支持正则（`egrep`）、固定字符串（`fgrep`）
- **`sort`**：排序，支持按字段排序（`-k`）和指定分隔符（`-t`）
- **`tar`**：归档工具，常配合 `gzip` / `bzip2` 压缩

## Shell 父子关系

执行脚本或命令时，Shell 会创建**子进程**运行。命令列表 `(cmd1; cmd2)` 在子 Shell 中执行。后台模式（`&`）让命令异步运行——通过 `jobs` 查看后台任务。

> [!note] 内建命令 vs 外部命令
> 外部命令（如 `/bin/ps`）创建子进程；内建命令（如 `cd`、`echo`、`history`）不创建子进程，效率更高。

## 环境变量

- **全局变量**：`export` 导出，子 shell 可继承但修改不影响父 shell
- **局部变量**：仅在当前 shell 可见
- **PATH**：命令搜索路径，`:` 分隔
- **数组**：`arr=(a b c)`，`${arr[0]}` 访问单个元素，`${arr[*]}` 访问全部

## 进程管理

通过**进程信号**与进程通信：

| 信号 | 含义 |
|------|------|
| 2 (INT) | 中断 (Ctrl+C) |
| 9 (KILL) | 无条件终止 |
| 15 (TERM) | 正常终止（默认） |
| 17 (STOP) | 暂停但不终止 |
| 19 (CONT) | 恢复运行 |

使用 `kill`（按 PID）或 `killall`（按进程名）发送信号。`top` 实时监控进程资源使用。

## 文件系统与权限

- **挂载**：`mount` / `umount`，永久挂载需配置 `/etc/fstab`
- **分区**：`fdisk` 交互式管理，分区后需用 `mkfs` 格式化
- **权限**：`chmod`（`[ugoa][+-=][rwx]`）、`chown`、`chgrp`
- **特殊位**：SUID（以所有者权限执行）、SGID（继承目录属组）、粘着位

## 用户管理

`useradd` / `userdel` / `usermod` 管理用户账户。`passwd` / `chpasswd` 管理密码。`groupadd` / `groupmod` 管理组。

## 关联页面

- [[sources/shell-scripting-摘要|Shell 脚本编程来源摘要]]
- [[concepts/Linux Shell环境|Linux Shell 环境]]
- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]]
- [[concepts/操作系统基础|操作系统基础]]
