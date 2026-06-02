---
title: "Linux Shell 环境完全指南"
date: 2026-06-02
tags: [linux, shell, bash, fish, zsh, terminal, neovim, tmux, package-manager]
type: tool
aliases: [Linux笔记, Shell命令, Bash教程, Fish配置, Zsh配置, 终端配置]
description: Linux Shell 环境完整指南：Bash 脚本编程基础（变量/条件/循环/函数/trap/文件测试/重定向/heredoc）、Fish Shell 安装与配置（语法/函数/alias/提示符）、Zsh + Oh My Zsh + Antigen、Shell 永久变量、终端多路复用(tmux基础)、Linux 代理配置脚本、lazygit/fzf/ripgrep/fd 工具安装、Neovim/LazyVim/插件生态、Pacman/APT 包管理、Alacritty 终端、PowerShell 升级、Linux 文件系统基础（权限/目录结构）
---

# Linux Shell 环境完全指南

## Bash 脚本编程基础

### Shebang 与注释

```bash
#!/bin/bash  # 指定解释器路径，必须正确
# 这是注释
```

### 变量

```bash
# 赋值（= 周围不能有空格）
name=value

# 使用命令输出赋值
now=$(date)
now=`date`   # 反引号等效

# 引用变量
echo ${name}     # 推荐写法
echo $name       # 字符串外部可直接使用
```

### 内部变量

| 变量 | 含义 |
|------|------|
| `$0` | 当前脚本文件名 |
| `$1` ~ `$n` | 第 n 个参数 |
| `$#` | 参数个数 |
| `$@` | 所有参数列表（数组形式） |
| `$*` | 所有参数列表（字符串形式） |
| `$?` | 上一条命令的退出状态（0=成功） |
| `$$` | 当前 shell 进程 ID |
| `$!` | 最后一个后台命令的进程号 |
| `$-` | 当前 shell 的选项标志 |

### 数组

```bash
# 普通数组
arr=(a b c d)
echo ${arr[0]}           # 单个元素
echo ${#arr[@]}          # 数组长度
echo ${arr[@]}           # 所有元素

# 关联数组（Bash 4.0+）
declare -A site
site["google"]="www.google.com"
site["runoob"]="www.runoob.com"
echo ${!site[@]}         # 所有键
echo ${site["google"]}   # 取值
```

### 字符串操作

```bash
# 长度
echo ${#string}

# 子串提取
echo ${STRING:1:3}       # 从位置1起3字符
echo ${STRING:12}        # 从位置12到末尾

# 替换（首个匹配）
echo ${STRING[@]/be/eat}

# 替换（全部匹配）
echo ${STRING[@]//be/eat}

# 匹配开头替换
echo ${STRING[@]/#to be/eat now}

# 匹配结尾替换
echo ${STRING[@]/%be/eat}

# 用命令结果作为替换内容
echo ${STRING[@]/%be/be on $(date +%Y-%m-%d)}

# 删除前缀/后缀
echo ${var#pattern}      # 最短匹配删除前缀
echo ${var##pattern}     # 最长匹配删除前缀
echo ${var%pattern}      # 最短匹配删除后缀
echo ${var%%pattern}     # 最长匹配删除后缀
```

### 条件判断

```bash
NAME="George"
if [ "$NAME" = "John" ]; then
  echo "John Lennon"
elif [ "$NAME" = "George" ]; then
  echo "George Harrison"
else
  echo "Others"
fi
```

#### 数字比较

| 比较符 | 含义 | 比较符 | 含义 |
|--------|------|--------|------|
| `-lt` | < | `-le` | <= |
| `-gt` | > | `-ge` | >= |
| `-eq` | == | `-ne` | != |

#### 字符串比较

| 比较符 | 含义 |
|--------|------|
| `=` / `==` | 相等 |
| `!=` | 不等 |
| `-z` | 为空（长度为0） |
| `-n` | 非空 |

#### 文件测试

| 测试符 | 含义 |
|--------|------|
| `-e` | 文件存在 |
| `-f` | 是普通文件 |
| `-d` | 是目录 |
| `-r` | 可读 |
| `-w` | 可写 |
| `-x` | 可执行 |
| `-s` | 文件非空 |
| `-nt` / `-ot` | 比另一个文件新/旧 |

#### 逻辑组合

```bash
if [[ $VAR_A[0] -eq 1 && ($VAR_B = "bee" || $VAR_T = "tee") ]]; then
    command...
fi
```

> **注意**：`[` 和 `]` 与条件表达式之间必须有空格。

#### case 语句

```bash
case "$variable" in
    "$condition1" )
        command...
    ;;
    "$condition2" )
        command...
    ;;
    *)
        default_command...
    ;;
esac
```

### 循环

```bash
# for 循环
for arg in [list]; do
    command...
done

# 遍历数组
for N in ${NAMES[@]}; do
    echo $N
done

# 遍历命令输出
for f in $(ls); do
    echo $f
done

# C 风格 for 循环
for ((i=0; i<10; i++)); do
    echo $i
done

# while 循环
while [ condition ]; do
    command...
done

# until 循环（条件为假时执行）
until [ condition ]; do
    command...
done
```

**break 与 continue**：

```bash
while [ $COUNT -lt 10 ]; do
  COUNT=$((COUNT+1))
  if [ $(($COUNT % 2)) = 0 ]; then
    continue  # 跳过偶数
  fi
  echo $COUNT
done
```

### 函数

```bash
function adder {
  echo "$(($1 + $2))"
}

adder 12 56   # 输出 68
```

参数通过 `$1`、`$2` ... `$n` 传递；`$@` 获取所有参数。

### 重定向与管道

```bash
# 输出重定向
command > file        # 覆盖写入
command >> file       # 追加写入

# 错误输出重定向
command 2> file       # 仅 stderr
command &> file       # stdout + stderr

# 输入重定向
command < file

# Heredoc（多行输入）
cat << EOF
多行
文本
EOF

# 管道
command1 | command2   # 将 stdout 传给下一个命令
command1 |& command2  # stdout + stderr 传给下一个命令
```

### Trap 命令

捕获信号，用于清理临时文件或优雅退出：

```bash
# 捕获 SIGINT（Ctrl+C）和 SIGTERM
trap "echo Booh!" SIGINT SIGTERM

# 脚本退出时清理临时文件
trap "rm -f /tmp/tempfile; exit" EXIT

# 忽略信号
trap '' SIGINT
```

### Here Document 与 Here String

```bash
# Here Document
cat << 'EOF' > config.txt
name=value
port=8080
EOF

# Here String
grep "pattern" <<< "$variable"
```

## Shell 环境变量（永久生效）

- 使用 `export` 并在 `~/.bashrc` 中 source
- 直接修改 `~/.bashrc` 并 source

```bash
export MY_VAR=value
source ~/.bashrc
```

### 常用环境变量

| 变量 | 含义 |
|------|------|
| `PATH` | 可执行文件搜索路径 |
| `HOME` | 用户主目录 |
| `USER` | 当前用户名 |
| `SHELL` | 当前 shell |
| `LANG` | 语言和编码设置 |
| `PS1` | 主提示符格式 |

## Fish Shell

### 安装（Ubuntu）

```shell
sudo apt-add-repository ppa:fish-shell/release-3
sudo apt update
sudo apt install fish
```

### 设为默认 Shell

```shell
chsh -s /usr/bin/fish
```

### 基本语法

Fish 语法设计自然直观，无需 `then`/`fi`/`do`/`done` 等 bash 关键字：

```fish
# if 语句
if grep fish /etc/shells
    echo Found fish
else if grep bash /etc/shells
    echo Found bash
else
    echo Got nothing
end

# switch 语句
switch (uname)
case Linux
    echo Hi Tux!
case Darwin
    echo Hi Hexley!
case '*'
    echo Hi, stranger!
end

# while 循环
while true
    echo "Loop forever"
end

# for 循环（支持管道迭代）
for file in *.txt
    cp $file $file.bak
end
```

### 命令替代

Fish 使用 `()` 而非 bash 的反引号：

```fish
echo (date)

for i in (ls)
    echo $i
end
```

### 函数与 Alias

Fish 用 `function` 替代 alias，定义存放在 `~/.config/fish/functions/` 中（`.fish` 后缀文件自动加载）：

```fish
# 定义函数
function ll
    ls -lhG $argv
end

# 重新定义命令时需加 command 前缀，防止无限递归
function ls
    command ls -hG $argv
end
```

Fish 3.0+ 推荐使用 `alias -s`：

```fish
alias -s l "ls -lah"
alias rmi "rm -i"

# 持久化 alias 到文件
funcsave rmi
```

执行 bash 脚本：

```fish
bash -c 'SomeBashCommand'
```

### 环境变量配置

配置文件位置：
- `~/.config/fish/config.fish` — 用户级
- `/etc/fish/config.fish` — 全局

```fish
# 设置 PATH
set -x PATH /opt/demo/bin /home/guest/bin $PATH
```

### 提示符自定义

```fish
function fish_prompt
    set_color purple
    date "+%m/%d/%y"
    set_color FF0
    echo (pwd) '>'
    set_color normal
end
```

### Web 配置界面

```fish
fish_config
```

浏览器自动打开 `localhost:8000`，可在线配置提示符和配色主题。

### 实用功能

- **历史命令搜索**：`Ctrl+R`
- **采纳建议**：`→`（右箭头），`Alt+→` 采纳部分
- **自动补全**：基于 man page 的上下文感知补全

## Zsh + Oh My Zsh

### 安装

```shell
sudo apt install zsh -y
sudo apt install curl -y

# 安装 Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# 安装 Antigen 插件管理器
curl -L git.io/antigen > antigen.zsh
```

### Oh My Zsh 常用插件

在 `~/.zshrc` 中配置：

```shell
plugins=(git docker extract zsh-autosuggestions zsh-syntax-highlighting)
```

## 终端多路复用

tmux 用于终端会话管理，支持分屏、会话保持、断线重连。

```shell
# 安装
sudo apt install tmux

# 基本操作
tmux new -s {session_name}    # 新建会话
tmux attach -t {session_name} # 重新连接
tmux ls                        # 列出会话
tmux kill-session -t {name}   # 关闭会话

# 窗格操作（Prefix 默认为 Ctrl+b）
Prefix %    # 垂直分割
Prefix "    # 水平分割
Prefix 方向键  # 切换窗格
Prefix x    # 关闭窗格
```

## Neovim 环境

### 安装最新版（Ubuntu）

```shell
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:neovim-ppa/unstable
sudo apt-get update
sudo apt-get install neovim

# Python 支持
sudo apt-get install python-dev python-pip python3-dev python3-pip
```

### Neovim 插件生态

**算法竞赛**：
- [competitest.nvim](https://github.com/xeluxee/competitest.nvim) — 测试用例管理
- [cphelper.nvim](https://github.com/jmerle/competitive-companion) — 从网页获取题目信息
- [leetcode.nvim](https://github.com/kawre/leetcode.nvim) — LeetCode 集成

**笔记**：
- [mkdnflow.nvim](https://github.com/jakewvincent/mkdnflow.nvim) — Markdown 工作流
- [neorg](https://github.com/nvim-neorg/neorg) — 结构化笔记系统

**大纲/符号**：
- [symbols-outline.nvim](https://github.com/simrat39/symbols-outline.nvim) — 代码符号大纲

参考：
- [Neovim 官方](https://github.com/neovim/neovim)
- [awesome-neovim](https://github.com/rockerBOO/awesome-neovim)

## 包管理器

### APT（Ubuntu/Debian）

```shell
sudo apt update                    # 更新软件包列表
sudo apt upgrade                   # 升级已安装软件包
sudo apt install {package}         # 安装
sudo apt remove {package}          # 卸载
sudo apt autoremove                # 清理无用依赖
sudo apt search {keyword}          # 搜索
```

### C++ 开发环境

```shell
sudo apt install build-essential gdb
sudo apt install cmake
```

### Pacman（Arch）

```shell
sudo pacman -Syu     # 全面更新
sudo pacman -S {pkg} # 安装
sudo pacman -R {pkg} # 卸载
```

## 高效命令行工具

### lazygit

终端 Git GUI，提供可视化提交界面：

```shell
LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
curl -Lo lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/latest/download/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz"
tar xf lazygit.tar.gz lazygit
sudo install lazygit /usr/local/bin
```

### fzf（模糊搜索）

命令行模糊搜索工具，支持 `Ctrl+R` 历史搜索、`Ctrl+T` 文件搜索、`**` 触发补全。

### ripgrep（rg）

极速递归文本搜索：

```shell
cargo install ripgrep
# 或
sudo apt install ripgrep

# 使用
rg "pattern" {path}
```

### fd

快速文件查找（`find` 的现代替代）：

```shell
cargo install fd-find
# 或
sudo apt install fd-find

# 使用
fd "pattern" {path}
```

## 代理配置

### Linux 代理脚本

```shell
#!/bin/sh
hostip=$(cat /etc/resolv.conf | grep nameserver | awk '{ print $2 }')
wslip=$(hostname -I | awk '{print $1}')
port=7890

PROXY_HTTP="http://${hostip}:${port}"

set_proxy(){
  export http_proxy="${PROXY_HTTP}"
  export HTTP_PROXY="${PROXY_HTTP}"
  export https_proxy="${PROXY_HTTP}"
  export HTTPS_proxy="${PROXY_HTTP}"
  export ALL_PROXY="${PROXY_SOCKS5}"
  export all_proxy=${PROXY_SOCKS5}

  git config --global http.https://github.com.proxy ${PROXY_HTTP}
  git config --global https.https://github.com.proxy ${PROXY_HTTP}

  echo "Proxy has been opened."
}

unset_proxy(){
  unset http_proxy
  unset HTTP_PROXY
  unset https_proxy
  unset HTTPS_PROXY
  unset ALL_PROXY
  unset all_proxy
  git config --global --unset http.https://github.com.proxy
  git config --global --unset https.https://github.com.proxy

  echo "Proxy has been closed."
}

test_setting(){
  echo "Host IP:" ${hostip}
  echo "WSL IP:" ${wslip}
  resp=$(curl -I -s --connect-timeout 5 -m 5 -w "%{http_code}" -o /dev/null www.google.com)
  if [ ${resp} = 200 ]; then
    echo "Proxy setup succeeded!"
  else
    echo "Proxy setup failed!"
  fi
}

if [ "$1" = "set" ]; then
  set_proxy
elif [ "$1" = "unset" ]; then
  unset_proxy
elif [ "$1" = "test" ]; then
  test_setting
else
  echo "Unsupported arguments."
fi
```

## Python 环境

### 安装指定版本 venv

```shell
# 确认 python3 版本（如 3.10）
python3 --version
sudo apt install python3.10-venv
python3 -m venv {name}

# 激活
source {name}/bin/activate
```

## SSH

```shell
# 安装 SSH 客户端和辅助工具
sudo apt install openssh-client sshpass wget

# 生成密钥
ssh-keygen -t ed25519 -C "comment"
```

## Alacritty 终端

Alacritty — GPU 加速的跨平台终端模拟器，支持 Windows/Linux/macOS。配置通过 `alacritty.toml` 文件，支持 live reload。

## PowerShell 升级（Windows）

```powershell
# 查看当前版本
$PSVersionTable.PSVersion

# 搜索可用版本
winget search Microsoft.PowerShell

# 安装稳定版
winget install --id Microsoft.Powershell --source winget

# 安装预览版
winget install --id Microsoft.Powershell.Preview --source winget
```

## Linux 文件系统基础

### 权限管理

```shell
# 数字方式（rwx = 4+2+1）
chmod 777 file       # rwxrwxrwx
chmod 755 file       # rwxr-xr-x

# 文字方式
chmod u=rwx,go=rx file
# u=user(所有者), g=group(组), o=others(其他), a=all(全部)
# = 精确设置, + 添加, - 移除
```

**权限区别**：
- **文件**：r=可读内容，w=可修改，x=可执行
- **目录**：r=可列出内容（ls），w=可增删文件，x=可进入目录（cd）

```shell
# 改变所有者
chown user:group file

# 改变所属组
chgrp group file
```

### 目录结构

| 目录 | 用途 |
|------|------|
| `/bin` | 基础可执行文件（ls, cp, cat 等） |
| `/boot` | 开机加载文件（内核、grub） |
| `/dev` | 设备文件 |
| `/etc` | 系统配置文件 |
| `/lib` | 共享函数库 |
| `/mnt` | 临时挂载点 |
| `/opt` | 第三方可选软件 |
| `/tmp` | 临时文件（重启可能清空） |
| `/var` | 系统运行中变化的数据（日志、缓存） |

**`/usr` 层级**：

| 目录 | 用途 |
|------|------|
| `/usr/bin` | 用户命令 |
| `/usr/lib` | 库文件 |
| `/usr/local` | 自行安装的软件（bin/lib 等子目录） |
| `/usr/sbin` | 非必需的系统命令 |
| `/usr/share` | 架构无关的共享数据 |
| `/usr/include` | C/C++ 头文件 |
| `/usr/src` | 源代码 |
