---
title: ECS 架构
type: concept
status: archived
archived-from: drafts/ECS架构学习.md
updated: 2026-05-05
tags: [gamedev, architecture, ecs, unity, dots]
---

# ECS 架构

ECS（Entity Component System）是一种数据导向的游戏架构模式，Unity DOTS 是其代表性实现。

## 核心概念

### Entity（实体）
轻量级 ID 句柄，本身不含数据和行为，只是一个标识符。可以理解为指向一组组件的"钥匙"。

### Component（组件）
纯数据结构，不含任何逻辑：
- 在 Unity ECS 中实现为 `IComponentData`（值类型）或 `IBufferElementData`（动态数组）
- 按 Archetype 连续存储在 Chunk 中，实现高缓存命中率

### System（系统）
所有逻辑所在。系统查询匹配特定组件组合的 Entity，对其组件数据进行操作：
- `SystemAPI.Query<T>()` — 遍历所有拥有 T 组件的 Entity
- `IJobEntity` — 利用 Job System 并行处理
- System 之间通过 `UpdateBefore`/`UpdateAfter` 属性控制执行顺序

## 为什么 ECS 更快

核心原因不是"组合优于继承"，而是**内存布局**：

| | GameObject/MonoBehaviour | ECS |
|------|------|------|
| 内存排布 | 对象分散在堆上，引用跳跃 | 同 Archetype 的数据连续存储在 Chunk 中 |
| 缓存命中 | 低（pointer chasing） | 高（连续遍历） |
| 并行化 | 受限于主线程 | 天然适配 Job System + Burst 编译 |

## ECS vs GameObject 选择指南

| 场景 | 推荐 |
|------|------|
| 大量同类实体（子弹、敌人、粒子） | ECS |
| 复杂交互逻辑（对话系统、任务链） | GameObject |
| 需要频繁物理计算 | ECS |
| UI、音频等系统集成 | GameObject |
| 已有大量 MonoBehaviour 代码的项目 | 渐进迁移，避免全部重写 |

## 调试技巧

- **Entity Debugger**（Window > DOTS > Entities）：查看运行中的 Entity 和组件数据
- **System Graph**：可视化 System 依赖关系
- **Profiler**：使用 Unity Profiler 的 DOTS 专用面板
- 给 Entity 添加 `NameComponent` 便于在 Debugger 中识别

## 相关概念

- 数据导向设计（Data-Oriented Design）
- Unity Job System + Burst Compiler
- [[A-star算法]] — 寻路系统可受益于 ECS 的并行化能力
