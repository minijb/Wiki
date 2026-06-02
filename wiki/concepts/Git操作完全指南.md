---
title: "Git 操作完全指南"
type: concept
updated: 2026-06-02
tags: [git, version-control, workflow, branching]
aliases: [Git操作, Git教程, Git命令, 版本控制]
---

# Git 操作完全指南

Git 分布式版本控制系统的完整操作知识体系。从环境配置到高级工作流，覆盖日常开发的全部场景。

## 核心体系

### 环境配置

`git init` 创建仓库 → `git config` 设置用户与代理 → SSH 密钥免密认证。全局配置优化包括别名（`co`/`br`/`ci`/`st`）、编辑器（`code --wait`）、换行符处理、默认分支名。

详见 [[sources/git-operations-guide-摘要|Git 操作来源摘要]]。

### 日常操作

```shell
git add {file} && git commit -m "msg"
git push origin main
git pull origin main
```

交互式暂存（`git add -p`）、`commit --amend` 修正最后提交、`git blame` 行级追溯。

### 分支策略

经典模型：**master**（发布）→ **develop**（开发）→ **feature/release/hotfix**（临时，完即删）。Git Flow 自动化管理：`git flow feature start/finish/publish`。

相对引用：`^`（父提交）、`~n`（向上 n 级）、`^2`（合并提交的第二父分支）。

### 合并三件套

| 策略 | 特点 | 何时用 |
|------|------|--------|
| **Merge** | 保留分支历史，`--no-ff` 可追溯 | 合并功能分支到主干 |
| **Rebase** | 线性历史，`-i` 交互式整理 | 本地提交整理、同步上游 |
| **Cherry-pick** | 精准复用单个/范围提交 | 跨分支移植特定修复 |

交互式 rebase 命令：`pick`（保留）、`reword`（改消息）、`squash`/`fixup`（合并）、`edit`（修改）、`drop`（删除）。

### 撤销体系

```
工作区 → git restore {file}
暂存区 → git restore --staged {file}
提交   → git reset --soft/--mixed/--hard
已推送 → git revert（反向提交）
救命   → git reflog（90 天后悔药）
```

### Stash 暂存

临时保存工作区和暂存区修改：`push/pop/apply/drop/clear`。冲突时推荐 `git stash branch` 分配到新分支。

### 高级工具

- **Submodule** — 子项目依赖，`add/update --remote/--init`，`clone --recurse-submodules`
- **Tag** — 发布标记，轻量标签 vs 附注标签（推荐，含签名和元信息）
- **Bisect** — 二分查找引入 bug 的提交
- **Hooks** — 事件钩子（`pre-commit`/`commit-msg`/`pre-push`）
- **Worktree** — 同时检出多分支到不同目录
- **Gitignore** — 白名单（`!`）+ 全局排除文件

## 关联页面

- [[sources/git-operations-guide-摘要|Git 来源摘要]]
- [[concepts/CSharp进程管理|C# 进程管理]] — dotnet CLI 工具
- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]] — Windows + WSL2 混合开发环境搭建
