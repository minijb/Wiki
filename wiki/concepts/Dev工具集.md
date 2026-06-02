---
title: "Dev 工具集"
type: concept
updated: 2026-06-02
tags: [docker, protobuf, dotnet, svn, obsidian, note-taking, tools]
aliases: [开发工具, Docker工具, Protobuf教程, SVN工具, 笔记系统]
---

# Dev 工具集

开发辅助工具合集：Docker 容器化、Protobuf 序列化、dotnet CLI、SVN 版本控制、Obsidian 笔记系统。覆盖从构建部署到知识管理的完整工具链。

## 核心体系

### Docker 容器化

三要素流程：**Dockerfile**（构建蓝图）→ `docker build` → **Image**（只读模板）→ `docker run` → **Container**（运行实例）。

```shell
docker build -t my-app .
docker run -d -p 8080:80 --name app my-app
docker-compose up -d          # 多容器编排
```

详见 [[sources/dev-tools-misc-摘要|Dev 工具集来源摘要]]。

### Protobuf 3

Google 的高效二进制序列化协议：

```proto
syntax = "proto3";
message UserInfo {
  string user_id = 1;          // 字段编号 1-15 占 1 字节
  repeated string members = 2; // 重复字段
  optional int32 status = 3;   // 可选字段
}
```

**字段标签**：`optional`（可设置/默认值）、`repeated`（重复/保留顺序）、`map`（键值对）。

**高级特性**：枚举（首值必须为 0）、`Any`（任意嵌入类型）、`oneof`（多选一共享内存）、`package`（命名空间）。

**C# 实战**：`ToByteArray()` 序列化 → `MergeFrom(bytes)` 反序列化。Unity 集成需裁剪 `TargetFrameworks`。

### dotnet CLI

```shell
dotnet new console -n MyApp    # 创建
dotnet build && dotnet run     # 构建运行
dotnet add package Newtonsoft  # 依赖管理
dotnet publish -c Release      # 发布
```

### SVN 版本控制

集中式版本控制，四大核心操作：

```
checkout（拉取）→ 修改 → update（同步）→ commit（提交）
                       ↓
              status（复查）→ revert（修复）→ resolve（解决冲突）
```

SVN vs Git：集中式（简单/需联网）vs 分布式（离线/轻量分支）。

### Obsidian 笔记系统

**目录原则**：按领域划分（非技术栈）、功能导向命名、≤3 层深度。

```
Notes/
├─ 0_Areas/（知识领域）
├─ 1_Projects/（项目独立）
├─ 3_References/（速查表/RFC/论文）
├─ 9_Inbox/（临时）
└─ 10_Archive/（归档）
```

**文件命名**：关键词前置（`设计模式_工厂模式.md`）、状态标识（`[WIP]`、`[DRAFT]`）、ISO 日期（`20230725_`）。

**插件**：Admonition（增强提示块）、Dataview（元数据聚合查询）。

**最佳实践**：`ripgrep` 全文搜索、定期归档（每季度）、避免深层嵌套（用标签关联）、`⭐_Highlights/` 精华内容。

### xmake

基于 Lua 的跨平台 C/C++ 构建工具，内置包管理。

## 关联页面

- [[sources/dev-tools-misc-摘要|Dev 工具集来源摘要]]
- [[concepts/CSharp进程管理|C# 进程管理]] — dotnet 相关
- [[concepts/Git操作完全指南|Git 操作指南]] — 版本控制对比
