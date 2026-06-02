---
title: 光线追踪入门-摘要
updated: 2026-06-02
tags: [raytracing, cpp, graphics, path-tracing]
source: raw/gamedev/rendering/ray-tracing-in-one-weekend.md
---

# 光线追踪入门 — 摘要

## 主题

基于 Peter Shirley 的 *Ray Tracing in One Weekend* 教程，从零用 C++ 实现一个完整的光线追踪器，涵盖光线求交、材质系统、反走样、相机模型和景深。

## 核心知识点

### 基础框架
- **vec3 类**：同时用于颜色（color）、空间点（point3）和向量
- **PPM 格式**输出图像（P3 ASCII）
- 随机数工具：`random_in_unit_sphere`（拒绝法）、`random_unit_vector`、`random_on_hemisphere`

### 光线与相机
光线方程 $P(t) = A + tb$。基础相机设置视口（宽高比 16:9）、像素网格（从左上角扫描）和光线发射方向。像素坐标 Y 轴朝下，与 3D 空间相反。

### 球体求交
二次方程 $t^2 a + 2th + c = 0$（简化形式），判别式 $h^2 - ac \ge 0$。取较小正根为最近交点。法线 = $(P-C)/r$。

### Hittable 抽象
`hittable` 基类 + `hit_record`（交点/法线/材质/正反面）。`set_face_normal` 判断内外，对玻璃等双面物体重要。

### 反走样
每个像素随机偏移采样（`samples_per_pixel`），取平均值。Gamma 校正：钳制到 [0, 0.999]。Shadow Acne 修复：`t_min = 0.001` 而非 0。

### 材质系统
- **Lambertian**：`scatter_direction = normal + random_unit_vector()`，半球漫反射
- **Metal**：$\mathbf{r} = \mathbf{v} - 2(\mathbf{v} \cdot \mathbf{n})\mathbf{n}$，模糊度 `fuzz` 控制
- **Dielectric**：折射 Snell 定律 + 全内反射判断 + Schlick 菲涅尔近似

### 折射与全内反射
$$R'_{\perp} = \frac{\eta}{\eta'}(R + \cos\theta \cdot \mathbf{n})$$
$$R'_{\parallel} = -\sqrt{1 - |R'_{\perp}|^2} \cdot \mathbf{n}$$
当 $\frac{\eta}{\eta'} \sin\theta > 1$ 时全内反射。

### 可定位相机与景深
- **FOV**：$\tan(\theta/2) = h / focal\_length$
- **定向**：lookfrom/lookat/vup → 正交基 (u, v, w)
- **景深**：随机偏移光线起点到镜头圆盘上

### 渲染流程
`ray_color` 递归：光线 → 击中 → 材质 scatter → 递归 × attenuation → 超过 max_depth 返回黑色。

## 相关概念

- [[光线追踪入门]] — 概念页面
- [[渲染管线理论]] — GAMES101 光线追踪与路径追踪理论
- [[Shader高级特性]] — Unity 阴影与渲染路径
