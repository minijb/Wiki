---
title: "现代 CMake 构建指南"
date: 2026-06-02
tags: [cmake, build, cpp, toolchain, vcpkg]
type: tool
aliases: [CMake最佳实践, CMake教程, CMake构建]
description: 现代 CMake 完整指南：target-based 构建、PUBLIC/PRIVATE/INTERFACE 可见性规则、变量与缓存、生成器表达式、configure_file、文件操作、vcpkg 集成
---

# 现代 CMake 构建指南

## 最小项目

```cmake
cmake_minimum_required(VERSION 3.10)
project(Tutorial VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(${PROJECT_NAME} main.cpp)
```

## CMake 构建流程

CMake 分为两个阶段：

| 阶段 | 工作 |
|------|------|
| **Configure** | 解析 CMakeLists.txt，展开变量，生成构建系统文件（Makefile/Ninja/VS Solution） |
| **Build** | 调用底层构建工具编译链接 |

### 命令行

```bash
# 选择生成器
cmake -G "Ninja" -S /path/to/source -B build

# 构建
cmake --build build --config Debug --target MyApp

# 单配置生成器（Makefile/Ninja）通过 CMAKE_BUILD_TYPE 指定
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release

# 多配置生成器（VS/Xcode）通过 --config 在构建时指定
cmake --build build --config Release
```

## 库：target-based 核心

### 创建库

```cmake
add_library(targetName [STATIC | SHARED | MODULE]
    [EXCLUDE_FROM_ALL]
    source1 [source2 ...]
)
```

| 关键字 | 产物 | 说明 |
|--------|------|------|
| `STATIC` | `.lib`(Win) / `.a`(Linux) | 静态链接 |
| `SHARED` | `.dll`(Win) / `.so`(Linux) | 运行时动态链接 |
| `MODULE` | 类似 SHARED | 动态加载（非链接时依赖） |

全局切换动态/静态库：

```cmake
set(BUILD_SHARED_LIBS YES)
# 或命令行：cmake -DBUILD_SHARED_LIBS=YES ..
```

> 除非有强烈理由，否则避免在 `add_library` 中硬编码 STATIC/SHARED，允许项目级切换。

### 目标命名建议

- 不需要以 `Lib` 开头
- 目标名称不需要与项目名称相同
- 使用描述性名称（如 `math_utils`、`network_core`）

## PUBLIC / PRIVATE / INTERFACE

这是现代 CMake 最重要的概念。控制依赖的传递方向。

```cmake
target_link_libraries(targetName
    <PRIVATE|PUBLIC|INTERFACE> item1 [item2 ...]
)

target_include_directories(targetName
    <PRIVATE|PUBLIC|INTERFACE> [items...]
)
```

### 语义

| 关键字 | 自身需要 | 使用者需要 | 场景 |
|--------|---------|-----------|------|
| `PRIVATE` | ✅ | ❌ | 纯内部实现依赖 |
| `PUBLIC` | ✅ | ✅ | 接口中暴露了依赖的类型 |
| `INTERFACE` | ❌ | ✅ | header-only 库的依赖 |

### 示例

```cmake
# collector 内部使用 algo 和 engine，使用者不需要知道
# collector 接口中暴露了 ui 的类型，使用者必须链接 ui
add_library(collector src1.cpp)
add_library(algo src2.cpp)
add_library(engine src3.cpp)
add_library(ui src4.cpp)

target_link_libraries(collector
    PUBLIC ui
    PRIVATE algo engine
)

add_executable(myApp main.cpp)
target_link_libraries(myApp PRIVATE collector)
# myApp 自动链接 ui（通过 PUBLIC 传递），但不链接 algo/engine
```

### 用途对比

| 模式 | 示例 |
|------|------|
| `PRIVATE` + `add_library` | 内部实现文件，仅该目标编译时需要 |
| `PUBLIC` + `add_library` | 公共头文件中 `#include` 了另一个目标 |
| `INTERFACE` + header-only 库 | 虚拟目标，仅传递编译选项和依赖 |

### 全局 vs 目标级命令

```cmake
# ❌ 旧式：全局污染
include_directories(include/)
link_libraries(mylib)

# ✅ 现代：目标精确控制
target_include_directories(myTarget PUBLIC include/)
target_link_libraries(myTarget PUBLIC mylib)
```

`target_*` 命令可精确控制每个目标的依赖，避免全局污染，解决循环依赖问题。

## INTERFACE 库与编译特性

```cmake
# 创建虚拟接口库，统一管理编译选项
add_library(project_compiler_flags INTERFACE)
target_compile_features(project_compiler_flags INTERFACE cxx_std_17)
target_compile_options(project_compiler_flags INTERFACE
    -Wall -Wextra -Wpedantic
)

# 各目标只需链接它
target_link_libraries(myApp PRIVATE project_compiler_flags)
target_link_libraries(myLib PRIVATE project_compiler_flags)
```

## 变量系统

### 普通变量

CMake 中所有值都是字符串，多个值用分号 `;` 分隔。

```cmake
set(myVar a b c)      # "a;b;c"
set(myVar a;b;c)      # "a;b;c"
set(myVar "a b c")    # "a b c"（含空格的一个字符串）
set(myVar a "b c")    # "a;b c"

# 变量展开
set(foo ab)           # foo = "ab"
set(bar ${foo}cd)     # bar = "abcd"
set(baz ${foo} cd)    # baz = "ab;cd"
set(bar ${notSetVar}) # bar = ""（未定义变量展开为空）
```

### 括号语法（类 Lua 多行字符串）

```cmake
set(multiLine [[
First line
Second line
]])

# 含 = 的版本，防止内容中的 ]] 提前关闭
set(shellScript [=[
#!/bin/bash
[[ -n "${USER}" ]] && echo "Have USER"
]=])
```

括号语法中变量**不展开**，适合脚本和正则表达式。

### 缓存变量 (Cache)

存储在 `CMakeCache.txt`，跨配置持久化，需手动修改才会变化。

```cmake
set(varName value... CACHE type "docstring" [FORCE])
```

| type | 说明 |
|------|------|
| `BOOL` | 布尔值，GUI 显示为复选框 |
| `STRING` | 文本字符串 |
| `FILEPATH` | 文件路径 |
| `PATH` | 目录路径 |
| `INTERNAL` | 内部变量，GUI 不显示 |
| `option()` | `BOOL` 的快捷语法 |

```cmake
# option 是 BOOL 缓存变量的便捷写法
option(USE_MYMATH "Use tutorial provided math implementation" ON)
# 等价于：
set(USE_MYMATH ON CACHE BOOL "Use tutorial provided math implementation")
```

### 环境变量

```cmake
set(ENV{PATH} "$ENV{PATH}:/opt/myDir")
```

仅在 CMake 运行时有效，结束后恢复。

### 常用内置变量

| 变量 | 含义 |
|------|------|
| `CMAKE_SOURCE_DIR` | 顶层 CMakeLists.txt 所在目录 |
| `CMAKE_BINARY_DIR` | 构建输出根目录 |
| `CMAKE_CURRENT_SOURCE_DIR` | 当前 CMakeLists.txt 所在目录 |
| `CMAKE_CURRENT_BINARY_DIR` | 当前对应的构建输出目录 |
| `PROJECT_SOURCE_DIR` | 最近 `project()` 的源目录 |
| `PROJECT_BINARY_DIR` | 最近 `project()` 的构建目录 |
| `PROJECT_NAME` | 最近 `project()` 的名称 |

## configure_file

将输入文件复制到输出位置，并替换其中的 CMake 变量。

```cmake
configure_file(input output [@ONLY])
```

### 模板变量替换

```
// foo.h.in
#define FOO_VERSION_MAJOR @Foo_VERSION_MAJOR@
#define FOO_VERSION_MINOR @Foo_VERSION_MINOR@
#cmakedefine FOO_ENABLE
#cmakedefine FOO_STRING "@FOO_STRING@"
```

```cmake
# CMakeLists.txt
project(Foo VERSION 2.1)
option(FOO_ENABLE "Enable Foo" ON)
if(FOO_ENABLE)
    set(FOO_STRING "foo")
endif()
configure_file(foo.h.in foo.h @ONLY)
```

生成的 `foo.h`：

```cpp
#define FOO_VERSION_MAJOR 2
#define FOO_VERSION_MINOR 1
#define FOO_ENABLE
#define FOO_STRING "foo"
```

若 option 为 OFF，`#cmakedefine` 变为 `/* #undef FOO_ENABLE */`。

配合 `target_include_directories` 使用：

```cmake
target_include_directories(myTarget PUBLIC "${CMAKE_CURRENT_BINARY_DIR}")
```

## Option 与条件判断

```cmake
option(USE_MYMATH "Use custom math" ON)

if(USE_MYMATH)
    target_compile_definitions(MathFunctions PRIVATE "USE_MYMATH")
    add_library(SqrtLibrary STATIC mysqrt.cxx)
    target_link_libraries(MathFunctions PRIVATE SqrtLibrary)
else()
    target_link_libraries(MathFunctions PRIVATE std::sqrt_lib)
endif()
```

代码中使用条件编译：

```cpp
#ifdef USE_MYMATH
#  include "mysqrt.h"
#endif

double sqrt(double x) {
#ifdef USE_MYMATH
    return detail::mysqrt(x);
#else
    return std::sqrt(x);
#endif
}
```

## 生成器表达式 (Generator Expressions)

在 Configure 阶段设置表达式，Build 阶段才求值。

```cmake
# 检测编译器类型
set(gcc_like "$<COMPILE_LANG_AND_ID:CXX,ARMClang,AppleClang,Clang,GNU,LCC>")
set(msvc "$<COMPILE_LANG_AND_ID:CXX,MSVC>")

# 按编译器设置不同警告选项
target_compile_options(project_flags INTERFACE
    "$<${gcc_like}:-Wall;-Wextra;-Wshadow>"
    "$<${msvc}:-W3>"
)
```

### 常用形式

| 形式 | 说明 |
|------|------|
| `$<condition:true_string>` | 条件为真时返回 true_string |
| `$<IF:cond,true,false>` | 条件三目 |
| `$<BUILD_INTERFACE:...>` | 仅在构建当前项目时生效 |
| `$<INSTALL_INTERFACE:...>` | 仅在安装后作为外部库使用时生效 |

可嵌套：

```cmake
$<${msvc}:$<BUILD_INTERFACE:-W3>>
```

## 文件操作

### GLOB 文件查找

```cmake
# 非递归
file(GLOB SRC_FILES "src/*.cpp" "src/*.h")

# 递归子目录
file(GLOB_RECURSE ALL_SRC "src/**/*.cpp" "src/**/*.h")

# 相对于指定路径
file(GLOB_RECURSE SRCS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "src/**/*.cpp")
```

**注意**：`GLOB` 在新增/删除文件后不会自动更新，需要重新运行 CMake。添加 `CONFIGURE_DEPENDS` 可在构建时自动检测文件变化（CMake 3.12+）。

### 文件复制

```cmake
# 方法 1：configure_file（纯复制）
configure_file(config.ini ${CMAKE_CURRENT_BINARY_DIR}/config.ini COPYONLY)

# 方法 2：file(COPY)
file(COPY config.ini DESTINATION ${CMAKE_CURRENT_BINARY_DIR})
```

## 子目录

```cmake
add_subdirectory(source_dir [binary_dir] [EXCLUDE_FROM_ALL])
```

子目录的 CMakeLists.txt 自动继承父目录变量（非缓存变量）。

## 生成器选择

```bash
cmake -G -h  # 列出所有可用生成器
```

| 生成器 | 特点 |
|--------|------|
| Ninja | 极速增量构建，推荐 |
| Unix Makefiles | 经典，广泛支持 |
| Visual Studio 17 | Windows IDE 集成 |
| MSYS Makefiles | MinGW/MSYS2 环境 |
| Xcode | macOS IDE 集成 |

## compile_commands.json

用于 clangd、ccls 等语言服务器的代码补全和跳转：

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
# 或命令行：cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
```

构建后在 `build/` 目录生成 `compile_commands.json`，通常符号链接到项目根目录：

```bash
ln -s build/compile_commands.json .
```

## vcpkg 集成

vcpkg 是 C/C++ 包管理器，与 CMake 深度集成。

### 基本用法

```bash
# 安装
git clone https://github.com/Microsoft/vcpkg.git
cd vcpkg
./bootstrap-vcpkg.sh  # Linux/macOS
# or: bootstrap-vcpkg.bat  # Windows

# 安装包
vcpkg install fmt boost-asio

# 集成到 CMake
cmake -S . -B build \
    -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake
```

### CMakeLists.txt 中使用

```cmake
find_package(fmt CONFIG REQUIRED)
find_package(Boost REQUIRED COMPONENTS system)

target_link_libraries(myApp PRIVATE
    fmt::fmt
    Boost::system
)
```

安装 vcpkg 包后，`find_package` 自动找到。

## CMake 完整项目模板

```
project/
├── CMakeLists.txt          # 顶层
├── src/
│   ├── CMakeLists.txt      # add_library + 内部源文件
│   └── ...
├── include/
│   └── project/
│       └── header.h
├── tests/
│   ├── CMakeLists.txt      # add_executable + CTest
│   └── test_main.cpp
└── cmake/
    └── FindXXX.cmake       # 自定义 Find 模块
```

顶层 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.16)
project(MyProject VERSION 1.0.0 LANGUAGES CXX)

# 全局选项
option(BUILD_TESTS "Build tests" ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 接口库：统一编译选项
add_library(project_options INTERFACE)
target_compile_features(project_options INTERFACE cxx_std_17)

# 子项目
add_subdirectory(src)

if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

## 参见

- 官方文档：https://cmake.org/cmake/help/latest/
- vcpkg：https://github.com/Microsoft/vcpkg
