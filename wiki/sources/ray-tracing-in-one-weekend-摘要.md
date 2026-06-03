---
title: 光线追踪入门-摘要
updated: 2026-06-02
tags: [raytracing, cpp, graphics, path-tracing]
source: raw/gamedev/rendering/ray-tracing-in-one-weekend.md
---

# 光线追踪入门 — 摘要

## 主题

基于 Peter Shirley 的 *Ray Tracing in One Weekend* 教程，从零用 C++ 实现一个完整的路径追踪器（Path Tracer），涵盖光线求交、场景管理、材质系统、反走样、Gamma 校正、可定位相机和景深效果。

## 核心知识点

### 基础框架
- **vec3 类**：同时用于颜色（color）、空间点（point3）和向量，通过类型别名区分语义
- **PPM 格式**输出图像（P3 ASCII）
- 随机数工具：`random_double`、`random_in_unit_sphere`（拒绝法）、`random_unit_vector`、`random_on_hemisphere`、`random_in_unit_disk`
- **Interval 类**：管理合法的光线参数范围（t_min, t_max）
- **Gamma 校正**：`linear_to_gamma(linear) = sqrt(linear)`，在 Gamma 2 空间存储使亮度过渡平滑

### 光线与相机
光线方程 $P(t) = A + tb$。基础相机设置视口（宽高比 16:9）、像素网格（从左上角扫描）和光线发射方向。像素坐标 Y 轴朝下，与 3D 空间相反。光线从相机发射到场景（光路可逆）。

### 球体求交
二次方程简化形式 $t^2 a - 2th + c = 0$（$h = d \cdot (C-A)$），判别式 $h^2 - ac \ge 0$。简化公式 $t = (h \pm \sqrt{h^2-ac})/a$。取较小正根为最近交点。法线 = $(P-C)/r$。

### Hittable 抽象与场景管理
`hittable` 基类 + `hit_record`（交点/法线/材质/正反面）。`hittable_list` 线性扫描场景，每次找到更近交点就收缩搜索范围。`set_face_normal` 判断内外，对玻璃等双面物体重要。

### 反走样与 Shadow Acne
每像素多重采样（`samples_per_pixel`），采样点在像素内随机偏移 `[-0.5, 0.5]^2`，取平均值。Shadow Acne 修复：`t_min = 0.001` 而非 0。

### 漫反射
- **均匀半球采样**：随机半球方向 + 递归 ×0.5
- **True Lambertian**：散射方向 = `normal + random_unit_vector()`，分布正比于 $\cos\theta$，更真实
- 递归深度限制（max_depth），超过 → 黑色

### 材质系统
- **Lambertian**：`scatter_direction = normal + random_unit_vector()`，albedo 控制反射率
- **Metal**：$\mathbf{r} = \mathbf{v} - 2(\mathbf{v} \cdot \mathbf{n})\mathbf{n}$，模糊度 `fuzz` 控制 + 表面下方吸收
- **Dielectric**：折射 Snell 定律 + 全内反射判断 + Schlick 菲涅尔近似

### 折射与全内反射
$$R'_{\perp} = \frac{\eta}{\eta'}(R + \cos\theta \cdot \mathbf{n}),\quad R'_{\parallel} = -\sqrt{1 - |R'_{\perp}|^2} \cdot \mathbf{n}$$
当 $\frac{\eta}{\eta'} \sin\theta > 1$ 时全内反射。空心玻璃球通过双层球体（外折射率 1.50，内 1.00/1.50）实现。

### 可定位相机与景深
- **FOV**：$\tan(\theta/2) = viewport\_height / (2 \cdot focus\_dist)$
- **定向**：lookfrom/lookat/vup → 正交基 (u, v, w)，w 与视线反向
- **景深**：薄透镜近似，随机偏移光线起点到镜头圆盘上，所有光线聚焦于焦平面

### 渲染流程
`ray_color` 递归：光线 → 击中 → 材质 scatter → 递归 × attenuation → 超过 max_depth 返回黑色。背景为 Y 方向渐变（天空色）。

### 后续扩展
多线程渲染（OpenMP）、运动模糊、BVH 加速结构、纹理映射、体积渲染、重要性采样、混合密度。

## 相关概念

- [[光线追踪入门]] — 概念页面
- [[渲染管线理论]] — GAMES101 路径追踪与辐射度量学理论
- [[Shader高级特性]] — Unity 阴影系统与渲染路径
