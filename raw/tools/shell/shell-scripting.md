---
title: "Shell 脚本编程完全指南"
type: source
updated: 2026-06-02
tags: [shell, bash, linux, scripting, cli, process, filesystem, security]
aliases: [Shell编程, Bash脚本, Linux Shell]
description: Shell 脚本编程完整指南：基础命令与过滤器、Shell 父子关系与后台模式、环境变量与数组、用户与权限管理、文件系统与分区操作
---

# Shell 脚本编程完全指南

## 1. 基础命令与过滤器

### 1.1 ls — 文件列表

**通配符过滤器**：

| 符号 | 含义 |
|------|------|
| `?` | 匹配一个字符 |
| `*` | 匹配多个字符 |
| `[abc]` | 匹配指定范围，如 `[a-i]`、`[1-9]` |
| `[!abc]` | 排除指定范围，如 `[!a]` |

**常用选项**：

| 选项 | 含义 |
|------|------|
| `-i` | 覆盖前提示确认 (prompt before overwrite) |
| `-r` / `-R` | 递归操作 (recursive) |

### 1.2 链接：ln

```bash
# 软链接（符号链接）
ln -s {target} {link_name}

# 硬链接
ln {target} {link_name}
```

> [!note]
> 硬链接共享同一 inode，删除原文件不影响硬链接；软链接存储目标路径，原文件删除后断链。

### 1.3 sort — 排序

默认从小到大排序。

```bash
sort -t ':' -k 3 -n /etc/passwd
```

| 选项 | 含义 |
|------|------|
| `-k` | 指定排序字段 |
| `-t` | 指定分隔符 |
| `-n` | 按字符串数值排序 |
| `-r` | 反向排序 |

### 1.4 grep — 按行搜索

```bash
grep {opt} pattern {file}
```

| 选项 | 含义 |
|------|------|
| `-v` | 反向搜索（排除匹配行） |
| `-n` | 显示所在行号 |
| `-c` | 显示匹配个数 |
| `-e` | 多个匹配模式，如 `grep -e t -e f file` |

衍生工具：`egrep` 支持扩展正则表达式，`fgrep` 支持固定字符串搜索。

### 1.5 压缩与归档

**gzip** — 单文件压缩。

**tar** — 归档工具：

```bash
# 格式: tar function {options} obj1 obj2 ...

# 常用 function:
# -c  创建新的 tar 文件
# -x  提取归档文件
# -t  列出归档内容
# -A  将一个 tar 追加到另一个

# 常用 options:
# -f file  输出到文件
# -v       显示处理文件
# -C dir   切换到指定目录
# -z       通过 gzip 压缩
# -j       通过 bzip2 压缩

# 常用组合:
tar -czvf archive.tar.gz dir/    # 归档 + gzip 压缩
tar -xzvf archive.tar.gz         # 解压 + 提取
tar -cjvf archive.tar.bz2 dir/   # 归档 + bzip2 压缩
tar -xjvf archive.tar.bz2        # 解压 + 提取
```

> [!warning] 解压行为
> 如果归档时包含一个文件夹，解压后恢复为文件夹；如果归档的是多个文件，解压后直接展开为多个文件。`-z` / `-j` 放在选项最前面。

---

## 2. 进程管理

### 2.1 ps — 进程查看

`ps` 默认显示当前控制台下当前用户的进程。

| 选项 | 含义 |
|------|------|
| `-e` | 显示所有进程 |
| `-f` | 完整格式输出 |
| `-l` | 长格式输出 |

**长格式输出列**：
- **S**：进程状态
- **PRI**：进程优先级

### 2.2 top — 实时进程监控

| 字段 | 含义 |
|------|------|
| PR | 进程优先级 |
| VIRT | 虚拟内存占用量 |
| RES | 物理内存占用量 |
| SHR | 共享内存总量 |
| S | 进程状态 |

### 2.3 进程信号与通信

进程之间通过**进程信号**进行通信，进程先识别信号再决定是否接收。

| 信号编号 | 信号名 | 含义 |
|----------|--------|------|
| 1 | HUP | 挂起 |
| 2 | INT | 中断 |
| 3 | QUIT | 结束运行 |
| 9 | KILL | 无条件终止 |
| 11 | SEGV | 段错误 |
| 15 | TERM | 尽可能终止（默认） |
| 17 | STOP | 无条件停止但不终止 |
| 18 | TSTP | 停止/暂停，后台继续运行 |
| 19 | CONT | 在 STOP/TSTP 后恢复运行 |

**发送信号**：

```bash
# kill — 按 PID 发送信号
kill {options} {PID}
kill -s HUP {PID}    # 指定信号发送

# killall — 按进程名发送，支持通配符
killall http*
```

---

## 3. Shell 的父子关系

### 3.1 子 Shell

在父 shell 中执行 `bash` 命令会生成一个新的 shell 程序——**子 Shell**，在其中运行当前命令。运行 shell 脚本同理，会创建一个子进程。

```bash
ps -f
# 显示两个进程：一个父进程，一个运行 ps 的子进程
```

### 3.2 命令列表

```bash
# 非命令列表（在当前 shell 执行）
ls ; xxx ; xxx

# 命令列表（创建子 shell 执行）
(ls ; xxx ; xxx)
```

### 3.3 后台模式

```bash
# 在命令后加 & 放入后台
command &

# 返回：后台作业编号 + PID
# 查看当前终端的后台程序（其他终端不可见）
jobs

# 将进程列表放入后台
(ls; tar -czf archive.tar.gz dir; echo "done") &
```

### 3.4 协程

在后台生成一个子进程，在子 shell 内运行命令：

```bash
coproc sleep 10

# 扩展语法（注意 {} 前后有空格）
coproc MyTask { sleep 10; }
```

### 3.5 内建命令 vs 外部命令

```bash
which ps
# /bin/ps  ← 外部命令（文件系统命令，常位于 /bin）

# 外部命令会创建子进程运行
# 内建命令（如 cd、echo、history）不需要子进程
```

**history**：

```bash
history        # 查看历史命令
!42            # 执行编号 42 的历史命令
!!             # 执行上一条命令
```

**alias**：

```bash
alias -p                # 查看所有别名
alias li='ls -li'       # 设置别名
```

---

## 4. 环境变量

### 4.1 查看变量

```bash
env               # 查看全局环境变量
printenv HOME     # 查看指定变量
echo $HOME        # 同上
set               # 查看所有变量（含局部）
```

### 4.2 用户定义变量

```bash
one=Hello
echo $one

# 注意：
# 1. 等号两边不能有空格
# 2. 值包含空格需加引号
# 3. 子 shell 中设置的局部变量父 shell 无法访问
```

### 4.3 全局变量

```bash
val="hello"
export val        # 将局部变量提升为全局变量
```

> [!warning] 子 Shell 修改不影响父 Shell
> 子 shell 可以修改导出的全局变量，但修改**不影响父 shell 中的值**。无法通过 export 改变父 shell 中全局变量的值。

### 4.4 删除变量

```bash
unset val         # 不加 $ 符号
```

全局变量同理——在子进程中删除不影响父 shell。

### 4.5 PATH 环境变量

定义命令和程序的查找目录，使用 `:` 分隔：

```bash
PATH=$PATH:/home/user/scripts
# 通常加入 . 将当前目录加入搜索路径
```

### 4.6 非交互式 Shell

子 shell 可以继承父 shell 通过 `export` 导出的变量，但**不能继承局部变量**（设置但未导出的变量）。

### 4.7 数组变量

```bash
mytest=(one two three)

echo $mytest          # one（默认输出第一个元素）
echo ${mytest[2]}     # three
echo ${mytest[*]}     # one two three（所有元素）

# 直接通过索引修改
mytest[2]=eleven

# 删除
unset mytest          # 删除整个数组
unset mytest[2]       # 删除指定索引的值
```

---

## 5. 磁盘与文件系统

### 5.1 mount — 管理挂载设备

```bash
mount                # 查看已挂载设备
# 输出: 媒体文件名、挂载点、文件类型、访问状态

# 手动挂载
mount -t {type} {device} {directory}
# 例: mount -t vfat /dev/sdb1 /media/disk

# -o 指定挂载选项: ro(只读), rw(读写), user, loop...
```

**卸载**：

```bash
umount {directory|device}
# 注意：设备正在使用时卸载会失败
```

### 5.2 df — 磁盘空间

```bash
df          # 查看设备磁盘空间
df -h       # 易读格式（GB/MB）
```

### 5.3 du — 文件/目录大小

```bash
du {options} {target}
# 默认当前目录下所有文件

# 常用选项:
# -c  显示列出的文件总大小
# -h  易读格式
# -s  每个输出参数的总计
```

### 5.4 分区管理

```bash
# 查看分区
fdisk -lu {target}               # 无 target 则查看所有分区

# 交互式分区
sudo fdisk /dev/sdb
# 常用命令:
#   p  查看详细信息
#   n  添加新分区
#     e  扩展分区（逻辑分区）
#     o  主分区
#   w  保存并退出
```

### 5.5 创建文件系统与挂载

```bash
# 格式化（创建文件系统）
sudo mkfs.ext4 /dev/sdb1

# 临时挂载
sudo mkdir /mnt/my_partition
sudo mount -t ext4 /dev/sdb1 /mnt/my_partition

# 永久挂载：添加到 /etc/fstab
```

> [!warning]
> `mount` 命令只能临时挂载。若需开机自动挂载，必须将条目添加到 `/etc/fstab` 文件中。

### 5.6 逻辑卷管理 (LVM)

LVM 在物理磁盘之上创建抽象层，允许动态调整分区大小、跨磁盘扩展等。

---

## 6. 用户与权限安全

### 6.1 用户管理

**查看默认值**：

```bash
useradd -D
```

**创建用户**：

```bash
useradd -m {name}     # -m 同时在 HOME 中创建用户目录
```

| 选项 | 含义 |
|------|------|
| `-e {date}` | 过期日期，YYYY-MM-DD 格式 |
| `-g {group}` | 指定 GID 或组名 |
| `-G {group...}` | 附加组（可多个） |
| `-n` | 创建与登录名同名的新组 |
| `-p` | 指定密码 |
| `-u` | 指定 UID |
| `-s` | 指定默认 shell |

**修改默认值**：`useradd -D` 后加选项覆盖。

**删除用户**：

```bash
userdel -r test    # -r 同时删除用户 HOME 目录
```

**修改用户 — usermod**：

| 选项 | 含义 |
|------|------|
| `-l` | 修改登录名 |
| `-L` | 锁定账户（禁止登录） |
| `-U` | 解除锁定 |
| `-p` | 修改密码 |

**密码管理**：

```bash
passwd {user}             # 交互式修改密码
chpasswd < users.txt      # 批量修改密码（root 用户）
# users.txt 格式: userid:password
```

**其他账户工具**：

| 命令 | 用途 |
|------|------|
| `chsh -s /bin/bash user` | 修改登录 shell（必须用完整路径） |
| `chfn` | 修改备注字段（交互式） |
| `chage` | 管理用户密码有效期 |

`chage` 选项：
- `-E`：密码过期日期
- `-I`：密码过期到账户锁定的天数
- `-m`：修改密码的最少间隔天数
- `-W`：过期前多少天提示

日期格式：`YYYY-MM-DD` 或从 1970-01-01 起算的天数。

### 6.2 组管理

```bash
groupadd {name}                      # 创建组（默认不添加用户）

# 将用户加入组
usermod -G {groupname} {username}    # 注意：-G 是附加组，-g 是主组！
```

> [!warning] `-g` vs `-G`
> `usermod -g` 修改**主组**（primary group），`usermod -G` 修改**附加组**（supplementary groups）。两者行为完全不同。

```bash
groupmod -g {GID} {groupname}        # 修改 GID
groupmod -n {newname} {oldname}       # 修改组名
```

### 6.3 文件权限

```bash
ls -l
# 输出示例: -rwxr-xr-- 1 user group 1024 Jan 1 12:00 file.txt
```

**第一个字符（文件类型）**：

| 字符 | 类型 |
|------|------|
| `-` | 普通文件 |
| `d` | 目录 |
| `l` | 符号链接 |
| `c` | 字符型设备 |
| `b` | 块设备 |
| `n` | 网络设备 |

**chmod — 修改权限**：

```bash
# 数字模式
chmod 755 file      # rwxr-xr-x

# 符号模式: [ugoa][+-=][rwx]
chmod u+x file      # 给 owner 加执行权限
chmod go-w file     # 移除 group 和 other 的写权限
chmod a=r file      # 所有人只读
```

**chown — 修改归属**：

```bash
chown {owner}[:{group}] {file}
```

**chgrp — 修改组**：

```bash
chgrp {group} {file}
```

### 6.4 特殊权限位

| 权限位 | 说明 |
|--------|------|
| **SUID** | 以文件所有者的权限执行（`chmod u+s`） |
| **SGID** | 以文件所属组的权限执行；目录下新文件继承目录的组（`chmod g+s`） |
| **粘着位 (sticky)** | 只有文件所有者才能删除文件（`chmod +t`，如 `/tmp`） |

```bash
chmod u+s file    # 设置 SUID
chmod u-s file    # 移除 SUID
chmod g+s dir     # 设置 SGID
chmod g-s dir     # 移除 SGID
```

> [!tip] SGID 的实际用途
> 在共享目录上设置 SGID 后，该目录下创建的所有新文件自动归属于目录的属组——每个用户创建的文件都属于同一组，实现文件共享。

---

## 参见

- [[concepts/Linux Shell环境|Linux Shell 环境]] — 概念综合页
- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]] — Windows 下的 Linux 环境
- [[concepts/操作系统基础|操作系统基础]] — 进程、内存、文件系统
