---
title: "现代 CMake 构建"
type: concept
updated: 2026-06-02
tags: [cmake, build, cpp, toolchain]
aliases: [CMake最佳实践, CMake构建]
---

# 现代 CMake 构建

现代 CMake 以 target 为基本单元，通过 PUBLIC/PRIVATE/INTERFACE 精确控制依赖传递。告别全局 `include_directories`/`link_libraries` 的混乱。

## Target-Based 核心原则

每个 target（可执行文件或库）独立声明自己的属性：

```cmake
target_link_libraries(app PRIVATE libA libB)   # 链接依赖
target_include_directories(libA PUBLIC include/)  # 头文件搜索路径
target_compile_features(libA PUBLIC cxx_std_17)   # C++ 标准要求
target_compile_definitions(libA PRIVATE DEBUG_MODE)  # 编译宏
```

这些属性的传递方向由可见性关键字控制。

## PUBLIC / PRIVATE / INTERFACE

这是 CMake 中最重要的设计：

- **PRIVATE**：仅目标自身编译时使用，依赖不传递
- **PUBLIC**：目标自身 + 依赖此目标的其他目标都使用
- **INTERFACE**：目标自身不使用，但依赖此目标的其他目标使用

```cmake
# collector 内部用了 algo/engine（使用者不需要）
# collector 的接口暴露了 ui 的类型（使用者必须也链接 ui）
target_link_libraries(collector
    PUBLIC  ui
    PRIVATE algo engine
)
```

**INTERFACE 库**是虚拟目标——不编译任何源文件，纯粹传递编译选项和依赖。统一管理 C++ 标准、警告选项等。

## 变量系统

CMake 中所有值都是字符串，多个值用分号分隔。三类变量：

- **普通变量**：`set(VAR value)`，仅在当前作用域有效
- **缓存变量**：`set(VAR value CACHE STRING "doc")`，持久化到 `CMakeCache.txt`
- **环境变量**：`set(ENV{PATH} ...)`，仅在 CMake 运行时有效

`option(USE_FEATURE "desc" ON)` 等价于 `BOOL` 类型的缓存变量。

## configure_file

在配置阶段将模板文件中的 CMake 变量替换为实际值，生成配置头文件：

```
#cmakedefine FEATURE_ENABLED    → #define FEATURE_ENABLED 或 /* #undef */
#define VERSION "@PROJECT_VERSION@"  → #define VERSION "1.0.0"
```

## 生成器表达式

`$<condition:value>` 在 Build 阶段求值（而非 Configure 阶段），用于按编译器、配置类型等条件设置编译选项。

## vcpkg 集成

通过 `CMAKE_TOOLCHAIN_FILE` 引入 vcpkg，安装的包可通过标准的 `find_package` 使用，无需手动配置 include/lib 路径。

## 构建流程速查

```bash
# 配置
cmake -G Ninja -S . -B build -DCMAKE_BUILD_TYPE=Release

# 构建
cmake --build build --config Release

# 安装
cmake --install build --prefix /usr/local
```

## 关联页面

- [[sources/cmake-guide-摘要|现代 CMake 构建 来源摘要]]
