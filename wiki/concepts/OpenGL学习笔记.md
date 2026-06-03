---
title: OpenGL学习笔记
updated: 2026-06-02
tags: [opengl, glsl, graphics, rendering]
aliases: [LearnOpenGL, OpenGL基础]
---

# OpenGL学习笔记

基于 LearnOpenGL 教程的 OpenGL 渲染编程实践笔记。

## OpenGL 状态机模型

OpenGL 本质上是一个状态机，通过 Context 管理所有状态。Object 是对状态子集合的抽象：

```
glGen*() → glBind*() → 配置 → glBind*(0) 解绑
```

扩展（Extensions）与硬件绑定，需运行时检测可用性（`if (GL_ARB_extension_name)`）。

## 缓冲对象体系

### 三者关系

```
VBO: 存储顶点数据（位置、法线、UV等）
EBO: 存储索引（减少重复顶点）
VAO: 存储所有 attribute pointer 配置 + VBO/EBO 引用
```

### VBO Usage 模式

| 模式 | 场景 |
|:-----|:-----|
| `GL_STATIC_DRAW` | 网格不变（建筑） |
| `GL_DYNAMIC_DRAW` | 网格会变（变形） |
| `GL_STREAM_DRAW` | 每帧更新（粒子） |

### 顶点属性配置

`glVertexAttribPointer(index, size, type, normalized, stride, offset)` — 每个顶点属性从**当前绑定到 `GL_ARRAY_BUFFER` 的 VBO** 中获取数据。

### 完整配置流程

```c++
// 初始化
glBindVertexArray(VAO);
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, (void*)0);
glEnableVertexAttribArray(0);
// 解绑
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindVertexArray(0);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);  // 在 VAO 之后解绑！

// 每帧渲染
glUseProgram(shader);
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, count, GL_UNSIGNED_INT, 0);
```

> [!warning] 绑定顺序注意
> EBO 必须在 VAO 之后解绑，因为 EBO 引用直接存储在 VAO 中。VAO 存 attribute pointer 数组，可随时解绑 VBO。

## NDC 与帧缓冲

标准化设备坐标 (NDC)：中心 (0,0)，范围 (-1,-1) 到 (1,1)。`glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT)` 清除缓冲。帧缓冲可绑定多个渲染目标（MRT）。

## GLSL 速查

### 数据类型

| 类型 | 说明 |
|:-----|:-----|
| `vec2/3/4` | float 向量 |
| `ivecn`/`uvecn`/`bvecn`/`dvecn` | int/uint/bool/double 向量 |
| `mat3/mat4` | 矩阵 |
| `sampler2D` | 2D 纹理采样器 |
| `samplerCube` | 立方体贴图 |
| `sampler2DShadow` | 2D 阴影贴图 |
| `sampler2DArray` | 2D 纹理数组 |

分量访问：`.xyzw`（位置）/`.rgba`（颜色）/`.stpq`（纹理），支持混合重组。

### 内置函数

| 函数 | 用途 |
|:-----|:-----|
| `normalize(x)` | 单位化 |
| `dot(x, y)` | 点乘 |
| `cross(x, y)` | 叉乘 |
| `reflect(I, N)` | 反射方向 |
| `refract(I, N, eta)` | 折射方向 |
| `mix(a, b, t)` | 线性混合 |
| `clamp(x, a, b)` | 钳制 |
| `smoothstep(e0, e1, x)` | Hermite 平滑插值 |
| `step(edge, x)` | 阶跃函数 |

### 常用内置变量

- `gl_Position`：顶点着色器输出的裁剪空间坐标（vec4）
- `gl_FragCoord`：片元着色器输入的屏幕坐标
- `gl_FragDepth`：片元着色器可写的深度值
- `gl_FrontFacing`：是否是正面（bool）
- `gl_PointSize`：点精灵大小（float）

### Uniform 设置

`glUniform*` 后缀规则：`f`（1 float）、`3f`（3 float）、`fv`（float 数组）、`i`（int）。Uniform 值在程序对象中保持，直到重置或更新。

### 着色器间数据传递

`in`/`out` 关键字。顶点着色器的输出必须与片元着色器的输入**名称和类型完全匹配**。顶点属性至少 16 个位置，用 `layout(location=N)` 指定。

## 着色器编译流程

```
glCreateShader(GL_VERTEX_SHADER)
  → glShaderSource + glCompileShader
  → glGetShaderiv(GL_COMPILE_STATUS) 检查编译
glCreateShader(GL_FRAGMENT_SHADER)
  → glShaderSource + glCompileShader
glCreateProgram
  → glAttachShader(vert) + glAttachShader(frag)
  → glLinkProgram
  → glGetProgramiv(GL_LINK_STATUS) 检查链接
glUseProgram(program)
  → glDeleteShader(vert)  // 链接后可删除着色器对象
  → glDeleteShader(frag)
```

## 渲染管线

```
顶点着色器 → 曲面细分 → 几何着色器 → 裁剪 → 光栅化 → 片元着色器 → 帧缓冲
```

每个 Pass：指定物体 + MVP + 帧缓冲 + 输入/输出纹理 + 着色器。帧缓冲可绑定多个渲染目标（MRT）。

## 调试

- **RenderDoc**：跨平台截帧分析，查看 DrawCall/纹理/Shader 变量
- **Nsight Graphics**：NVIDIA 专用，支持 Shader 调试
- `glGetError()`：运行时错误码轮询
- `glGetShaderInfoLog()`：着色器编译错误详情
- `glGetProgramInfoLog()`：程序链接错误详情

### 常见排查
- **黑屏**：检查着色器编译/链接、VAO 绑定、VBO 数据
- **编译失败**：查看 info log，检查 GLSL 语法和 `#version`
- **链接失败**：检查 in/out 名称和类型匹配
- **EBO 不工作**：EBO 必须在 VAO 绑定时绑定，在 VAO 之后解绑

## 相关页面

- [[Unity Shader基础]] — HLSL/CG 对比 GLSL
- [[渲染管线理论]] — GPU 管线原理
- [[Shader高级特性]] — Unity 渲染路径
- [[Unity性能优化]] — GPU 数据管理与渲染优化
- [[learn-opengl-notes-摘要]] — 原始来源摘要
