---
title: 渲染管线理论-摘要
updated: 2026-06-02
tags: [graphics, rendering-pipeline, games101, raytracing, geometry]
source: raw/gamedev/rendering/rendering-pipeline-theory.md
---

# 渲染管线理论 — 摘要

## 主题

基于 GAMES101、GAMES104、GAMES202 课程的系统性图形学笔记，覆盖从数学基础、变换与光栅化、着色与纹理、几何处理、光线追踪到辐射度量学与全局光照的完整知识体系。

## 核心知识点

### 数学基础
齐次坐标（点 w=1，向量 w=0）、点乘（求夹角/投影）、叉乘（判断左右/内外）、矩阵运算（组合、逆、转置）。

### 变换管线
Model → View → Projection（MVP）。正交投影（平行线保持平行）、透视投影（近大远小，将 Frustum 挤压为长方体）。FOV 与视口尺寸的关系。

### 光栅化与反走样
采样法判断像素中心是否在三角形内。加速：包围盒、增量遍历。走样原因 = 采样频率低于信号频率。反走样 = 先模糊（低通滤波）再采样。MSAA/FXAA/TAA/DLSS 四种现代技术。

### Z-Buffer
每像素存最小深度值。O(n) 复杂度，与绘制顺序无关。不能处理透明物体。浮点精度导致 Z-Fighting。

### 着色模型
Blinn-Phong 反射模型：$L = L_a + L_d + L_s$。着色频率：Flat（面）→ Gouraud（顶点）→ Phong（像素）。重心坐标用于三角形内插值。

### 纹理与 Mipmap
双线性插值（放大）、Mipmap（缩小：预生成多级纹理，三线性插值）、各向异性过滤（解决非正方形区域模糊）、EWA 过滤。

### 几何处理
贝塞尔曲线（de Casteljau 算法，凸包性质）、Loop/Catmull-Clark 曲面细分、基于二次误差度量的网格简化。

### 光线追踪
光线方程 $P(t) = O + td$。加速结构：AABB → 均匀网格 → KD-Tree → BVH（按物体划分，最常用）。Shadow Mapping：从光源视角渲染深度图并比较。

### 辐射度量学
关键量：Radiant Flux → Intensity → Irradiance → Radiance。BRDF 描述表面反射特性。渲染方程 = 自发光 + 半球入射光积分。

### 路径追踪
蒙特卡洛积分 + 俄罗斯轮盘赌 + 直接光照重要性采样。高级方法：双向路径追踪（BDPT）、光子映射、MLT。

### 材质
菲涅尔效应（掠射角反射率高，Schlick 近似）、微表面模型（粗糙度由法线分布描述）、各向异性 BRDF、次表面散射（BSSRDF）。

### 动画与物理
质点弹簧系统、正向/逆向运动学（IK）、数值积分（欧拉/中点/RK4/Verlet）。

### 引擎架构
GAMES104 五层结构：Tool → Function → Resource → Core → Platform。

## 相关概念

- [[渲染管线理论]] — 概念页面
- [[Unity Shader基础]] — ShaderLab 基础
- [[Shader高级特性]] — Unity 渲染路径与阴影
- [[光线追踪入门]] — Ray Tracing in One Weekend
