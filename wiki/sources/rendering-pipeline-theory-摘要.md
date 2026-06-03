---
title: 渲染管线理论-摘要
updated: 2026-06-02
tags: [graphics, rendering-pipeline, games101, raytracing, geometry]
source: raw/gamedev/rendering/rendering-pipeline-theory.md
---

# 渲染管线理论 — 摘要

## 主题

基于 GAMES101、GAMES104、GAMES202 课程的系统性图形学笔记，覆盖从数学基础、变换与光栅化、着色与纹理、几何处理、光线追踪、辐射度量学与全局光照、材质与外观到动画物理、引擎架构和面试要点的完整知识体系。

## 核心知识点

### 数学基础
齐次坐标（点 w=1，向量 w=0）、点乘（求夹角/投影）、叉乘（判断左右/内外）、矩阵运算（组合、逆、转置）。

### 变换管线
Model → View → Projection（MVP）。正交投影（平行线保持平行）、透视投影（近大远小）。View Transform：相机在原点看向 -Z。FOV 关系：$\tan(fovY/2) = (h/2)/near$。

### 光栅化与反走样
采样法判断像素中心是否在三角形内（三次叉乘）。加速：包围盒、增量遍历。Z-Buffer：O(n) 复杂度，与绘制顺序无关，不能处理透明物体。反走样：先低通滤波再采样。MSAA/FXAA/TAA/DLSS/SMAA 五种技术。

### 着色模型
Blinn-Phong 反射模型：环境光 + 漫反射（Lambert 余弦定律）+ 高光（半程向量，指数 p）。着色频率：Flat（面）→ Gouraud（顶点）→ Phong（像素）。

### 纹理与 Mipmap
双线性插值（放大）、Mipmap（缩小：预生成 ≈1.33× 存储，三线性插值）、各向异性过滤（解决非正方形区域模糊）、EWA 过滤。应用：AO、凹凸贴图、法线贴图、位移贴图、3D 纹理、阴影贴图。

### GPU 管线架构
经典管线：VS → Tessellation → GS → Clipping → Primitive Assembly → Rasterization → FS。Mesh Shader 管线替代传统流程。现代 GPU SIMD/SIMT 架构，按 Render Pass 组织帧缓冲。

### 几何处理
贝塞尔曲线（de Casteljau 算法，凸包性质）、Loop/Catmull-Clark 曲面细分、基于二次误差度量的网格简化。

### 光线追踪
光线方程 $P(t) = O + td$。加速结构：AABB → 均匀网格 → KD-Tree → BVH（按物体划分，最常用）。Shadow Mapping：从光源视角渲染深度图比较。

### 辐射度量学
Radiant Flux → Intensity → Irradiance → Radiance。BRDF：$f_r = dL_r/dE_i$。渲染方程：自发光 + 半球入射光积分。路径追踪：蒙特卡洛积分 + 俄罗斯轮盘赌 + 直接光照重要性采样。高级：BDPT、MLT、Photon Mapping。

### 材质
菲涅尔效应（Schlick 近似）、微表面模型（粗糙度 = 法线分布）、各向异性 BRDF、次表面散射（BSSRDF）、参与介质。

### 动画与物理
质点弹簧系统、正向/逆向运动学（IK）、数值积分（欧拉/中点/RK4/Verlet/Position-Based）。

### 前向 vs 延迟渲染
Forward：逐对象逐光源，支持 MSAA 和半透明，多光源差。Deferred：G-Buffer + Lighting Pass，不支持 MSAA 和半透明，多光源好。现代趋势：Forward+ 与 Clustered Rendering。

### 引擎架构
GAMES104 五层结构：Tool → Function → Resource → Core → Platform。

### 面试要点
渲染管线顺序、顶点/片元着色器输入输出、Mipmap 原理、碰撞检测（AABB/OBB/SAT/DCD/CCD）。

## 相关概念

- [[渲染管线理论]] — 概念页面
- [[Unity Shader基础]] — ShaderLab 基础
- [[Shader高级特性]] — Unity 渲染路径与阴影
- [[光线追踪入门]] — Ray Tracing in One Weekend
- [[OpenGL学习笔记]] — OpenGL 渲染编程
