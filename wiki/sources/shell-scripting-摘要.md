---
title: "Shell 脚本编程 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [shell, bash, linux, scripting]
source: "raw/tools/shell/shell-scripting.md"
---

# Shell 脚本编程 — 摘要

来源：`raw/tools/shell/shell-scripting.md`

## 概述

Shell 脚本编程完整指南，涵盖基础命令与过滤器（ls/grep/sort/tar）、Shell 父子关系与后台模式（子进程/命令列表/协程）、环境变量与数组、进程管理（信号/kill/top）、磁盘与文件系统（mount/fdisk/mkfs）、用户与权限安全管理。

## 要点

- **过滤器**：`ls` 通配符、`grep` 搜索、`sort` 排序、`tar` 归档配合 gzip/bzip2
- **后台模式**：`&` 放入后台，`jobs` 查看，命令列表 `(cmd; cmd)&` 后台批量执行
- **进程信号**：KILL(9) 强制终止、TERM(15) 正常终止、STOP(17)/CONT(19) 暂停恢复
- **环境变量**：`export` 导出全局变量，子 shell 继承但修改不影响父 shell；PATH 用 `:` 分隔
- **数组**：`arr=(a b c)`，`${arr[*]}` 获取全部元素，`unset` 删除
- **分区流程**：`fdisk` 分区 → `mkfs.ext4` 格式化 → `mount` 挂载（永久需 `/etc/fstab`）
- **权限**：`chmod [ugoa][+-=][rwx]` 或数字模式；SUID/SGID/粘着位三种特殊权限
- **用户管理**：`useradd -m` 创建、`userdel -r` 删除（含 HOME）、`usermod -G` 加组
- **安全**：`-g` 修改主组，`-G` 修改附加组——两者行为完全不同

## 关联页面

- [[concepts/Shell脚本编程|Shell 脚本编程]] — 概念综合页
- [[concepts/Linux Shell环境|Linux Shell 环境]]
- [[concepts/操作系统基础|操作系统基础]]
