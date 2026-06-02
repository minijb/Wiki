---
title: "Git 操作完全指南 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/tools/git/git-operations-guide.md"
tags: [git, version-control, workflow]
---

# Git 操作完全指南

## 来源

`raw/tools/git/git-operations-guide.md` — Git 完整操作指南：仓库初始化、SSH/代理配置、远程操作、分支策略、合并（merge/rebase/cherry-pick）、stash、tag、submodule、撤销恢复、配置优化、bisect、hooks、reflog

## 要点

1. **环境初始化** — `git init`、`git config`（user.name/email）、代理设置（http/socks5）、SSH 密钥生成与配置
2. **远程操作** — `git remote` 管理、`push/pull/fetch`、`--force-with-lease` 安全推送、删除远程分支
3. **分支策略** — master/develop/feature/release/hotfix 模型；Git Flow 工作流自动化
4. **合并三件套** — Merge（fast-forward/--no-ff/--squash）、Rebase（交互式 pick/squash/reword/drop/fixup/--onto）、Cherry-pick（单提交/范围拣选）
5. **Stash 暂存** — `push/pop/apply/drop/clear`、冲突处理（分配到新分支）
6. **撤销体系** — `reset`（--soft/--mixed/--hard）、`revert`（反向提交）、`restore`（文件级）、`reflog`（90天后悔药）
7. **高级工具** — `bisect` 二分定位 bug、`hooks` 自动化钩子、`worktree` 并行分支、`blame` 行级追溯

## 关联 Wiki 页面

- [[concepts/Git操作完全指南|Git 操作]] — 概念页
