---
title: LearnOpenGL学习笔记-摘要
updated: 2026-06-02
tags: [opengl, glsl, graphics, rendering]
source: raw/gamedev/rendering/learn-opengl-notes.md
---

# LearnOpenGL 学习笔记 — 摘要

## 主题

基于 LearnOpenGL 教程的实践笔记，涵盖 OpenGL 状态机模型、VBO/VAO/EBO 缓冲对象体系、GLSL 着色器编程与编译流程、完整绘制流程和调试方法。

## 核心知识点

### OpenGL 状态机模型
OpenGL 本质上是一个状态机，通过 Context 管理所有状态。OpenGL Object 是对状态子集合的抽象，使用模式：创建 → 绑定 → 配置 → 解绑。扩展（Extensions）与硬件绑定，需运行时检测可用性。

### VBO（顶点缓冲对象）
GPU 端顶点数据存储。`glBufferData` 的 usage 参数：`GL_STATIC_DRAW`（一次设置多次使用）、`GL_DYNAMIC_DRAW`（频繁修改）、`GL_STREAM_DRAW`（一次设置几次使用）。

### VAO（顶点数组对象）
存储所有顶点属性配置的容器。记录 `glVertexAttribPointer` 设置、`glEnableVertexAttribArray` 状态、关联的 VBO 和 EBO。渲染时只需 bind VAO。一个 VAO 可有多个 VBO 和 EBO。

### EBO（索引缓冲）
通过索引引用顶点，减少重复数据。`glDrawElements(GL_TRIANGLES, count, type, offset)`。EBO 直接存储在 VAO 中。

### 绑定/解绑顺序
VAO → VBO（设置 attribute）→ EBO → 解绑 VBO → 解绑 VAO → 解绑 EBO。EBO 必须在 VAO 之后解绑，因为 EBO 引用存储在 VAO 中。

### GLSL 语法
- `in`/`out`：着色器间数据传递，名称和类型必须完全匹配
- `uniform`：全局变量，所有阶段可访问，由 C++ 端通过 `glUniform*` 设置
- `layout(location=N)`：指定顶点属性输入位置，至少 16 个位置
- 向量类型：`vecn`/`ivecn`/`uvecn`/`bvecn`/`dvecn`
- 分量访问：`.xyzw`/`.rgba`/`.stpq`，支持混合重组
- 内置变量：`gl_Position`（裁剪空间位置）、`gl_FragCoord`、`gl_FragDepth`、`gl_FrontFacing`
- 常用内置函数：`normalize`、`dot`、`cross`、`reflect`、`refract`、`mix`、`clamp`、`smoothstep`
- 纹理采样器类型：`sampler2D`、`sampler3D`、`samplerCube`、`sampler2DShadow`、`sampler2DArray`

### 着色器编译流程
`glCreateShader` → `glShaderSource` + `glCompileShader` → `glCreateProgram` → `glAttachShader` + `glLinkProgram` → `glUseProgram`。链接后可 `glDeleteShader`。使用 `glGetShaderiv`/`glGetShaderInfoLog` 检查编译和链接状态。

### NDC 与帧缓冲
标准化设备坐标范围 (-1,-1) 到 (1,1)，中心为 (0,0)。每个 Pass 指定物体+MVP+帧缓冲+输入/输出纹理+着色器，帧缓冲可绑定多个渲染目标（MRT）。

### 完整绘制流程
初始化：创建 VAO → 填充 VBO → 填充 EBO → 设置 attribute pointer → 解绑。每帧渲染：`glUseProgram` → `glBindVertexArray` → `glDrawElements` → 解绑。

### 调试工具
RenderDoc（跨平台截帧）、Nsight Graphics（NVIDIA）、`glGetError` 错误检查。常见问题：黑屏（检查着色器+VAO）、编译失败（查看 info log）、链接失败（检查 in/out 匹配）、EBO 顺序错误。

## 相关概念

- [[OpenGL学习笔记]] — 概念页面
- [[Unity Shader基础]] — Unity ShaderLab vs OpenGL GLSL
- [[渲染管线理论]] — GPU 管线原理
