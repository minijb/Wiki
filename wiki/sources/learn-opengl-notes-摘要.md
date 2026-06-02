---
title: LearnOpenGL学习笔记-摘要
updated: 2026-06-02
tags: [opengl, glsl, graphics, rendering]
source: raw/gamedev/rendering/learn-opengl-notes.md
---

# LearnOpenGL 学习笔记 — 摘要

## 主题

基于 LearnOpenGL 教程的实践笔记，涵盖 OpenGL 状态机模型、VBO/VAO/EBO 缓冲对象体系、GLSL 着色器编程和完整绘制流程。

## 核心知识点

### OpenGL 状态机模型
OpenGL 本质上是一个状态机，通过 Context 管理所有状态。OpenGL Object 是对状态子集合的抽象，使用模式：创建 → 绑定 → 配置 → 解绑。

### VBO（顶点缓冲对象）
GPU 端顶点数据存储。`glBufferData` 的 usage 参数：`GL_STATIC_DRAW`（一次设置多次使用）、`GL_DYNAMIC_DRAW`（频繁修改）、`GL_STREAM_DRAW`（一次设置几次使用）。

### VAO（顶点数组对象）
存储所有顶点属性配置的容器。记录 `glVertexAttribPointer` 设置、`glEnableVertexAttribArray` 状态、关联的 VBO 和 EBO。渲染时只需 bind VAO。

### EBO（索引缓冲）
通过索引引用顶点，减少重复数据。`glDrawElements(GL_TRIANGLES, count, type, offset)`。

### 绑定/解绑顺序
VAO → VBO（设置 attribute）→ EBO → 解绑 VBO → 解绑 VAO → 解绑 EBO。EBO 直接存储在 VAO 中，解绑 VAO 前不要解绑 EBO。

### GLSL 语法
- `in`/`out`：着色器间数据传递，名称和类型必须匹配
- `uniform`：全局变量，所有阶段可访问，由 C++ 端通过 `glUniform*` 设置
- `layout(location=N)`：指定顶点属性输入位置
- 向量类型：`vecn`/`ivecn`/`uvecn`/`bvecn`/`dvecn`
- 分量访问：`.xyzw`/`.rgba`/`.stpq`
- 内置变量：`gl_Position`（裁剪空间位置）、`gl_FragColor`

### 着色器编译流程
`glCreateShader` → `glShaderSource` + `glCompileShader` → `glCreateProgram` → `glAttachShader` + `glLinkProgram` → `glUseProgram`。链接后可删除着色器对象。

### 调试工具
RenderDoc（跨平台截帧）、Nsight Graphics（NVIDIA）、`glGetError` 错误检查。

## 相关概念

- [[OpenGL学习笔记]] — 概念页面
- [[Unity Shader基础]] — Unity ShaderLab vs OpenGL GLSL
- [[渲染管线理论]] — GPU 管线原理
