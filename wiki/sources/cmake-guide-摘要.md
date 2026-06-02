---
title: "现代 CMake 构建指南 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [cmake, build, cpp, toolchain]
source: "raw/tools/ci-cd/cmake-guide.md"
---

# 现代 CMake 构建指南 — 摘要

来源：`raw/tools/ci-cd/cmake-guide.md`

## 概述

现代 CMake target-based 构建指南，从最小项目到完整模板，核心聚焦 PUBLIC/PRIVATE/INTERFACE 可见性规则、变量与缓存系统、生成器表达式、configure_file 配置头文件生成、以及 vcpkg 包管理集成。

## 要点

- **构建两阶段**：Configure（解析 CMakeLists.txt，展开变量，生成构建文件）→ Build（调用底层工具编译链接）
- **target-based 原则**：用 `target_link_libraries` / `target_include_directories` 替代全局 `link_libraries` / `include_directories`，解决循环依赖和全局污染
- **PUBLIC/PRIVATE/INTERFACE**：PRIVATE 仅自身需要，PUBLIC 接口暴露依赖类型，INTERFACE 仅传递（header-only 库）。这是 CMake 最重要的概念
- **INTERFACE 库**：虚拟目标，统一管理编译选项和特性（`target_compile_features` / `target_compile_options`）
- **变量系统**：普通变量（分号分隔的字符串）、缓存变量（存 CMakeCache.txt 持久化）、环境变量（运行时临时）。option() 是 BOOL 缓存变量的快捷语法
- **configure_file**：模板文件中的 `@VAR@` 和 `#cmakedefine` 在配置阶段被替换，生成带版本号和条件编译开关的头文件
- **生成器表达式**：`$<condition:value>` 在 Build 阶段求值，可按编译器类型设置不同编译选项
- **vcpkg 集成**：通过 `CMAKE_TOOLCHAIN_FILE` 指定 vcpkg.cmake，安装的包可通过 `find_package` 直接使用

## 关联页面

- [[concepts/现代CMake构建|现代 CMake 构建]] — 概念综合页
