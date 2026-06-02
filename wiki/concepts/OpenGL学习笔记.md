---
title: OpenGL学习笔记
updated: 2026-06-02
tags: [opengl, glsl, graphics, rendering]
aliases: [LearnOpenGL, OpenGL基础]
---

# OpenGL学习笔记

基于 LearnOpenGL 教程的 OpenGL 渲染编程实践笔记。

## OpenGL 对象模型

OpenGL 是状态机，通过 Context 管理状态。对象是对状态子集合的封装：

```
glGen*() → glBind*() → 配置 → glBind*(0) 解绑
```

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
> EBO 必须在 VAO 之后解绑，因为 EBO 引用直接存储在 VAO 中。

## GLSL 速查

### 数据类型

| 类型 | 说明 |
|:-----|:-----|
| `vec2/3/4` | float 向量 |
| `mat3/mat4` | 矩阵 |
| `sampler2D` | 2D 纹理采样器 |
| `samplerCube` | 立方体贴图 |

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

### 常用内置变量

- `gl_Position`：顶点着色器输出的裁剪空间坐标
- `gl_FragCoord`：片元着色器输入的屏幕坐标
- `gl_FragDepth`：片元着色器可写的深度值

## 着色器编译流程

```
glCreateShader(GL_VERTEX_SHADER)
  → glShaderSource + glCompileShader
glCreateShader(GL_FRAGMENT_SHADER)
  → glShaderSource + glCompileShader
glCreateProgram
  → glAttachShader(vert) + glAttachShader(frag)
  → glLinkProgram
glUseProgram(program)
// 链接后可 glDeleteShader
```

## 渲染管线

```
顶点着色器 → 曲面细分 → 几何着色器 → 裁剪 → 光栅化 → 片元着色器 → 帧缓冲
```

每个 Pass：指定物体 + MVP + 帧缓冲 + 输入/输出纹理 + 着色器。帧缓冲可绑定多个渲染目标（MRT）。

## 调试工具

- **RenderDoc**：跨平台截帧分析
- **Nsight Graphics**：NVIDIA 专用
- `glGetError()`：运行时错误检查

## 相关页面

- [[Unity Shader基础]] — HLSL/CG 对比 GLSL
- [[渲染管线理论]] — GPU 管线原理
- [[LearnOpenGL学习笔记-摘要]] — 原始来源摘要
