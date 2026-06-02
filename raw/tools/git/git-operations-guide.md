---
title: "Git 操作完全指南"
date: 2026-06-02
tags: [git, version-control, workflow, branching, merge]
type: tool
aliases: [Git操作, Git教程, Git速查, Git命令]
description: Git 完整操作指南：仓库初始化、SSH配置、代理设置、远程操作(push/pull/fetch/remote)、分支操作、合并策略(rebase交互式/merge/cherry-pick)、stash暂存、tag标签、submodule子模块、commit规范、撤销与恢复(reset/revert/restore)、gitignore白名单、相对引用、git-flow工作流、reflog、bisect、blame、hooks
---

# Git 操作完全指南

## 环境初始化

### 创建仓库与用户配置

```shell
# 创建 Git 仓库
git init

# 全局用户配置
git config --global user.name {name}
git config --global user.email {email}
```

### 设置代理

```shell
# HTTP/HTTPS 代理
git config --global http.proxy 127.0.0.1:7890
git config --global https.proxy 127.0.0.1:7890

# SOCKS5 代理
git config --global http.proxy socks5 127.0.0.1:7891
git config --global https.proxy socks5 127.0.0.1:7891

# 查看代理
git config --global --get http.proxy
git config --global --get https.proxy

# 取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy
```

### SSH 连接

GitHub SSH 配置地址：[GitHub SSH Settings](https://github.com/settings/keys)

```shell
# 生成 SSH 密钥（推荐 ed25519）
ssh-keygen -t ed25519 -C "your_email@example.com"
```

将生成的 `.pub` 公钥内容添加到 GitHub SSH Keys 页面。


### Git 配置优化

```shell
# 查看所有配置
git config --list

# 设置默认分支名为 main
git config --global init.defaultBranch main

# 设置别名
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.lg "log --graph --oneline --all --decorate"

# 设置默认编辑器
git config --global core.editor "code --wait"

# 大小写敏感 (Windows 上推荐)
git config --global core.ignorecase false

# 自动转换换行符
git config --global core.autocrlf input  # Linux/macOS
git config --global core.autocrlf true   # Windows

# 设置 pull 时默认使用 rebase 而非 merge
git config --global pull.rebase true
```

### .gitignore 全局配置

```shell
# 创建全局 gitignore
git config --global core.excludesfile ~/.gitignore_global
```

常用 `.gitignore_global` 模板：

```txt
# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# 编译产物
*.o
*.obj
*.exe
*.dll
*.so

# 依赖
node_modules/
__pycache__/
*.pyc
```

# 测试连接
ssh -T git@github.com
```

## 远程操作

### 远程仓库管理

```shell
# 查看远程仓库
git remote -v

# 添加远程仓库
git remote add origin {url}

# 修改远程仓库地址
git remote set-url origin {url}

# 删除远程仓库
git remote remove origin
```

### 拉取与推送

```shell
# 从远程拉取（不合并）
git fetch origin

# 从远程拉取并合并
git pull origin {branch}

# 推送到远程
git push origin {branch}

# 首次推送并设置上游
git push -u origin {branch}

# 强制推送（危险操作）
git push --force origin {branch}
git push --force-with-lease origin {branch}  # 更安全的强制推送

# 删除远程分支
git push origin --delete {branch}
```

## 提交与日志

```shell
# 添加文件到暂存区
git add {file/*}

# 交互式暂存
git add -p

# 提交
git commit
git commit -m {message}

# 修改最近一次提交（补充遗漏文件或修正 message）
git commit --amend
```

### 查看日志

```shell
git log
git log --oneline
git log --graph --oneline --all   # 图形化展示所有分支
git log -p {file}                 # 查看文件变更历史
git log --author="{name}"         # 按作者过滤
git log --since="2024-01-01"      # 按时间过滤
git log -S"{keyword}"             # 搜索引入/删除某关键词的提交
```

### Git Blame

查看文件每一行的最后修改者和提交：

```shell
git blame {file}
git blame -L 10,20 {file}   # 查看指定行范围
```

## 分支操作

### 基本分支命令

```shell
# 查看分支
git branch
git branch -a   # 包括远程分支
git branch -v   # 显示最后提交

# 创建分支
git branch {name}

# 切换分支
git switch {name}
git checkout {name}

# 创建并切换
git checkout -b {name}
git switch -c {name}

# 删除分支
git branch -d {name}
git branch -D {name}   # 强制删除

# 合并分支
git merge {name}
```

### 相对引用

相对引用简化 commit 的指定：

- `^` — 向上移动一个提交记录；`HEAD^^` 表示 HEAD 的父提交的父提交
- `~{num}` — 向上移动 num 个提交记录；`HEAD~3` 等同于 `HEAD^^^`
- `^` 在合并提交时可指定父分支：`HEAD^1` 是第一父提交，`HEAD^2` 是第二父提交

## 分支管理策略

### 主分支 master / main

存放正式发布版本。对于科研项目，可存放最佳指标的快照。

### 开发分支 develop

预发布或日常开发分支。合并时推荐使用 `--no-ff` 保留合并节点，保持提交历史可追溯。

### 临时分支

- **feature** — 功能开发
- **release** — 预发布准备
- **fixbug / hotfix** — 紧急修复

临时分支使用完毕后应删除。对于暂时搁置的功能：可先合并到 develop，再通过 rebase 在下一次提交中移除。

### Git Flow 工作流

参考：[Git Flow Cheatsheet](https://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)

```shell
# 初始化 git flow
git flow init

# Feature 分支
git flow feature start {name}
git flow feature finish {name}
git flow feature publish {name}   # 发布到远端
git flow feature pull origin {name}
git flow feature track {name}     # 跟踪远端特性

# Release 分支（finish 时合并到 master 和 develop，并在 master 打 tag）
git flow release start {版本号}

# Hotfix 分支（同理）
git flow hotfix start {版本号}
```

## 合并策略

### Merge 类型

- **fast-forward** — 线性历史无分叉时自动使用，不创建合并节点
- **recursive / ort**（默认）— 三方合并算法，创建合并节点
- **octopus** — 多分支合并
- **ours** — 保留当前分支内容
- **subtree** — 子树合并

### 快速合并（Fast-Forward）

两个分支一前一后无分叉时，直接移动 HEAD。**合并更改但不合并分支**。

使用 `--squash` 压缩所有更改到暂存区：

```shell
git merge --squash feature
git commit -m "merge feature"
```

### 非快速合并（--no-ff）

保留分支历史，创建合并节点：

```shell
git merge --no-ff feature
```

### 合并冲突解决

```shell
# 冲突后查看状态
git status

# 查看冲突文件
git diff

# 解决冲突后标记为已解决
git add {file}

# 继续合并
git merge --continue

# 放弃合并
git merge --abort
```

### Rebase（变基）

变基将分支的提交记录复制到另一个分支上，创造**线性**的提交历史。

```shell
# 先同步主干
git fetch {main_branch}

# 将当前分支变基到主分支之后
git checkout {feature_branch}
git rebase {main_branch}

# 等价于
git rebase {main_branch} {feature_branch}
```

变基后可选择 fast-forward merge 或 `--no-ff`。

**`--onto` 选项** 用于跨分支迁移提交：

```shell
git rebase --onto master dev next
```

选取在 `next` 但不在 `dev` 中的 commit，变基到 `master` 上。

### 交互式 Rebase

用于整理本地提交历史，打开编辑器后每个提交一行：

```shell
git rebase -i HEAD~{num}
```

**可用命令**：

| 命令 | 说明 |
|------|------|
| `pick` (p) | 保留该提交 |
| `reword` (r) | 保留提交，修改 message |
| `edit` (e) | 保留提交，暂停以修改内容 |
| `squash` (s) | 合并到前一个提交，保留两个 message |
| `fixup` (f) | 合并到前一个提交，丢弃 message |
| `drop` (d) | 删除该提交 |
| `break` (b) | 在此处暂停 |

**处理 rebase 冲突**：

```shell
# 解决冲突后
git add {file}
git rebase --continue

# 跳过当前提交
git rebase --skip

# 放弃 rebase
git rebase --abort
```

### Cherry-Pick（拣选）

从某个分支选取特定 commit 应用到当前分支：

```shell
git cherry-pick {commit1} {commit2}
```

连续提交范围 `A..B`（不含 A）：

```shell
git cherry-pick A..B
```

包含 A：

```shell
git cherry-pick A^..B
```

**适用场景**：在多分支间复用特定修改（如将 bug 修复从 feature 分支同步到 master）。

**冲突解决**：解决冲突后执行 `git cherry-pick --continue`。

## Stash（暂存）

将**工作区和暂存区**的修改暂存起来，使工作区恢复干净。

### 常用命令

```shell
# 暂存修改
git stash
git stash -m {name}
git stash -u   # 包含未跟踪文件
git stash -a   # 包含未跟踪和忽略文件

# 查看 stash 列表
git stash list
git stash show
git stash show -p   # 查看详细差异

# 取出 stash
git stash pop                    # 取出栈顶并删除
git stash apply                  # 取出栈顶但不删除
git stash pop --index {num}
git stash apply stash@{n}        # 取出指定 stash
git stash branch {branch_name} {index}  # 取出到新建分支

# 删除 stash
git stash drop [id]
git stash clear
```

### 处理冲突

多个 stash 或 stash 与当前工作区可能产生冲突：

- **预防**：将 stash 分配到新分支 `git stash branch`
- **解决**：修改冲突文件 → `git add .` → `git commit` → `git stash drop`（冲突不自动删除 stash）

## Tag（标签）

标签分两种：

- **轻量标签**：类似不会改变的分支，只是某个提交的引用指针
- **附注标签**：Git 数据库中的完整对象，包含打标签者信息、日期、注释，支持 GPG 签名验证

```shell
# 列出标签
git tag
git tag -l "v1.*"    # 通配符过滤

# 删除标签
git tag -d {tag_name}
git push origin --delete {tag_name}  # 删除远程标签

# 创建附注标签（推荐）
git tag -a {tag_name} [hash] -m {message}

# 创建轻量标签
git tag {tag_name} [hash]

# 推送标签
git push origin {tag_name}
git push origin --tags   # 推送所有标签
```

## Submodule（子模块）

### 添加子模块

```shell
git submodule add {url} {path}
```

### 更新子模块

- 在子目录中：`git pull`
- 在主目录中拉取最新：`git submodule update --remote [path]`
  - 无路径则更新所有子模块
  - 更新后主项目记录的 commit id 会变化，需提交主项目变更
- 按记录 id 更新：`git submodule update [path]`
  - 使子模块恢复到主项目记录的 commit id（不一定是最新 master）

### 克隆含子模块的仓库

**按需初始化**：

```shell
git clone {main}
git submodule init {path}
git submodule update {path}
# 合并写法
git submodule update --init [path]
```

**全部更新**：

```shell
git clone {main}
git submodule init
git submodule update
```

**一步到位**：

```shell
git clone --recurse-submodules {url}
```

## 撤销与恢复

### Reset — 移动 HEAD

```shell
# --soft: 移动 HEAD，保留暂存区和工作区（非破坏性）
git reset --soft HEAD~{num}/{hash}

# --hard: 移动 HEAD，重置暂存区和工作区（破坏性，不可逆）
git reset --hard HEAD~{num}/{hash}

# --mixed: 移动 HEAD，保留工作区，清空暂存区（默认行为）
git reset --mixed HEAD~{num}/{hash}
```

### 交互式撤销

```shell
git rebase -i HEAD~{num}/{hash}
```

### 恢复工作区文件

```shell
git checkout {file_name}
git checkout HEAD~{num}/{hash} {filename}
git checkout {branch_name} {filename}
```

### 恢复暂存区

```shell
git restore --staged {filename}   # 撤回暂存区，不影响工作区
git restore {filename}            # 将工作区恢复到上次提交状态
```

### Revert — 生成反向提交

```shell
git revert {hash}
git revert HEAD~{num}..HEAD   # 撤销一系列提交
```

生成一个"负修改"来中和目标 commit 的更改。适用于**撤销已推送的提交**（不能删除远程节点，只能追加反向节点）。

### Reflog — 救命的后悔药

记录 HEAD 和分支引用的所有变更历史（包括已删除的提交），默认保留 90 天：

```shell
# 查看历史
git reflog

# 恢复"丢失"的提交
git checkout HEAD@{n}
git branch recovered-branch HEAD@{n}
```

## Git Bisect（二分查找）

快速定位引入 bug 的提交：

```shell
# 开始二分查找
git bisect start

# 标记当前版本为 bad
git bisect bad

# 标记已知正常版本为 good
git bisect good {hash}

# Git 会自动切换到一个中间版本供测试
# 根据测试结果标记：
git bisect good    # 该版本正常
git bisect bad     # 该版本有问题

# 重复直到找到引入 bug 的提交，然后结束：
git bisect reset
```

## Git Hooks

Git 钩子位于 `.git/hooks/` 目录，在特定事件时自动执行脚本。

**常用钩子**：

| 钩子 | 触发时机 | 典型用途 |
|------|----------|----------|
| `pre-commit` | 提交前 | 代码格式检查、lint |
| `commit-msg` | 提交消息编辑后 | 校验 commit message 格式 |
| `pre-push` | 推送前 | 运行测试 |
| `post-checkout` | 切换分支后 | 安装依赖 |
| `post-merge` | 合并后 | 更新子模块 |

## Gitignore 与白名单

`!` 用于将忽略规则转为白名单：

```txt
*
!*/

# 跟踪此文件
!.gitignore

# 白名单跟踪 ./config/ 下所有内容
!config/
```

**常用 .gitignore 模式**：

```txt
# 编译产物
build/
*.o
*.exe

# IDE
.vscode/
.idea/

# 系统文件
.DS_Store
Thumbs.db

# 依赖
node_modules/
```

## Worktree（工作树）

同时检出多个分支到不同目录：

```shell
# 添加工作树
git worktree add ../feature-branch feature

# 列出工作树
git worktree list

# 移除工作树
git worktree remove ../feature-branch
```

适用场景：在构建过程中切换到其他分支修复 bug，或同时处理多个功能。

## 相关资源

- [Git 官方文档](https://git-scm.com/book/zh/v2)
- [Git Flow Cheatsheet](https://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)
- [Learn Git Branching](https://learngitbranching.js.org/)
- [Git飞行规则](https://github.com/k88hudson/git-flight-rules/blob/master/README_zh-CN.md)
