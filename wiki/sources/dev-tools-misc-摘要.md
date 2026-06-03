---
title: "Dev 工具集 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/tools/ci-cd/dev-tools-misc.md"
tags: [docker, protobuf, dotnet, svn, obsidian, note-taking]
---

# Dev 工具集：Docker Protobuf SVN dotnet 笔记系统

## 来源

`raw/tools/ci-cd/dev-tools-misc.md` — 开发工具合集：Docker 基本概念与常用命令、Protobuf 3 完整教程、静态博客工具（Docsify/Hugo/Hexo/VuePress/GitBook/Jekyll）、多端同步（Syncthing/Mobius Sync/Resilio Sync）、dotnet CLI、SVN 版本控制、Obsidian 笔记系统

## 要点

1. **Docker** — 三要素（Dockerfile→Image→Container）、`build/run/ps/stop/rm` 命令流、Dockerfile 基础指令（FROM/WORKDIR/COPY/RUN/CMD）、Docker Compose 多容器编排
2. **Protobuf** — Proto3 语法（syntax/message/字段编号）、字段标签（optional/repeated/map）、类型对照表（跨 C++/C#/Python/Go）、枚举（首值必须为 0）、Any/Oneof/Package、C# 完整示例（序列化/反序列化）、Unity 安装流程
3. **dotnet CLI** — 安装验证、`new/restore/build/run/publish/test` 命令、`add package/reference` 依赖管理
4. **SVN** — 集中式版本控制、四大核心操作（checkout/commit/update/revert）、生命周期（Update→修改→复查→修复→解决冲突→提交）、SVN vs Git 对比
5. **Obsidian 笔记系统** — 目录设计原则（按领域划分/功能导向/≤3层）、文件命名技巧（关键词前置/状态标识/ISO 日期）、嵌入（文件/图像/音频/视频/PDF/iframe/大小调整）、笔记别名、插件（Admonition/Dataview/Kanban/Templater/QuickAdd/Excalidraw 等 20+）、Mermaid 图表、最佳实践（检索增强/定期归档/避过度分类）
6. **xmake** — 基于 Lua 的跨平台 C/C++ 构建工具
7. **静态博客** — Docsify（运行时渲染，无需预构建）、Hugo（Go 极速构建）、Hexo（Node.js 中文社区）、VuePress（Vue 驱动）、GitBook（Git 仓库文档）、Jekyll（GitHub Pages）、Pelican（Python）、WordPress（动态 CMS）
8. **多端同步** — Syncthing（开源 P2P 端到端加密文件同步，支持 Win/Mac/Linux）、Mobius Sync（iOS 客户端）、Resilio Sync（闭源备选）

## 关联 Wiki 页面

- [[concepts/Dev工具集|Dev 工具集]] — 概念页
