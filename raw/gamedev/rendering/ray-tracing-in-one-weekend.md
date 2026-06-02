---
title: Ray Tracing in One Weekend 笔记
type: source
updated: 2026-06-02
tags:
  - raytracing
  - cpp
  - graphics
  - path-tracing
---

# Ray Tracing in One Weekend 笔记

> 基于 Peter Shirley 的系列教程，从零实现一个光线追踪器。

---

## 一、基础框架

### 1.1 PPM 图像输出

```cpp
std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";
// 逐像素输出 RGB 值
```

P3 格式表示 ASCII 编码的彩色 PPM 图像。

### 1.2 vec3 类

三维向量类是光线追踪器的核心数据结构。同时用于表示颜色（color = vec3）和空间点（point3 = vec3）。

```cpp
class vec3 {
    double e[3];
    // 基本运算：+、-、*、/、+=、*=、/=
    double length() const;
    double length_squared() const;
    static vec3 random();         // [0,1) 随机向量
    static vec3 random(min, max); // [min,max) 随机向量
    bool near_zero() const;       // 接近零向量判断
};

// 自由函数
double dot(const vec3& u, const vec3& v);
vec3 cross(const vec3& u, const vec3& v);
vec3 unit_vector(const vec3& v);
vec3 random_in_unit_sphere();
vec3 random_unit_vector();
vec3 random_on_hemisphere(const vec3& normal);
```

**random_in_unit_sphere**：用拒绝法在单位球内生成随机点。

**random_unit_vector**：标准化 random_in_unit_sphere。

**random_on_hemisphere**：如果随机方向与法线夹角 > 90°，则反向。

---

## 二、光线与相机

### 2.1 光线（Ray）

光线方程：$P(t) = A + tb$

- A：光线起点（origin）
- b：光线方向（direction），不必单位化
- t：参数（double），t > 0 表示光线前方

```cpp
class ray {
    point3 orig;
    vec3 dir;
    point3 at(double t) const { return orig + t * dir; }
};
```

### 2.2 基础相机

设置视口、像素网格和光线发射方向。

**关键参数：**
- 宽高比：`aspect_ratio = 16.0 / 9.0`
- 视口高度：`viewport_height = 2.0`
- 视口宽度：由图像宽高比推导
- 焦距：`focal_length = 1.0`（相机中心到视口平面的距离）
- 相机中心：原点 `(0, 0, 0)`

**像素网格：**
- 像素 (i, j) 从左上角开始，逐行向下扫描
- 图像坐标 Y 轴向下（与 3D 空间 Y 轴向上相反）
- 像素网格从视口边缘内缩半个像素间距

**光线发射：**
```cpp
ray get_ray(int i, int j) const {
    auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
    auto ray_direction = pixel_center - center;
    return ray(center, ray_direction);
}
```

---

## 三、球体相交

### 3.1 球体方程

$(C_x - x)^2 + (C_y - y)^2 + (C_z - z)^2 = r^2$

向量形式：$(C - P) \cdot (C - P) = r^2$

### 3.2 光线与球求交

代入光线方程：$(C - (A + td)) \cdot (C - (A + td)) = r^2$

展开为二次方程：$t^2(d \cdot d) - 2t(d \cdot (C-A)) + (C-A) \cdot (C-A) - r^2 = 0$

令：
- $a = d \cdot d$
- $h = d \cdot (C-A)$（简化后，原 $b = -2h$）
- $c = (C-A) \cdot (C-A) - r^2$

判别式：$h^2 - ac \ge 0$ 时有交点。

**简化公式**：
$$
t = \frac{h \pm \sqrt{h^2 - ac}}{a}
$$

取较小的正根为最近交点。

### 3.2 法线计算

球体上一点P的外向法线：$\mathbf{n} = (P - C) / r$（单位向量）。

---

## 四、Hittable 抽象与正面判断

### 4.1 Hittable 抽象类

```cpp
class hit_record {
    point3 p;            // 交点
    vec3 normal;         // 法线（始终指向光线）
    shared_ptr<material> mat;  // 材质指针
    double t;            // 光线参数
    bool front_face;     // 是否在物体外表面

    void set_face_normal(const ray& r, const vec3& outward_normal);
};

class hittable {
    virtual bool hit(const ray& r, double t_min, double t_max, hit_record& rec) const = 0;
};
```

### 4.2 正面/背面判断

```cpp
void set_face_normal(const ray& r, const vec3& outward_normal) {
    front_face = dot(r.direction(), outward_normal) < 0;
    normal = front_face ? outward_normal : -outward_normal;
}
```

- `front_face = true`：光线从外部射入（法线与光线反向）
- `front_face = false`：光线从内部射出（法线反转）
- 对玻璃球等内外两面的对象非常重要

### 4.3 球体实现

```cpp
class sphere : public hittable {
    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const {
        // 求解二次方程
        // 在 [t_min, t_max] 范围内找最近的根
        rec.t = root;
        rec.p = r.at(rec.t);
        vec3 outward_normal = (rec.p - center) / radius;
        rec.set_face_normal(r, outward_normal);
        rec.mat = mat;
        return true;
    }
};
```

---

## 五、反走样（Antialiasing）

### 5.1 多重采样

对每个像素发射多条随机偏移的光线，取颜色平均值：

```cpp
int samples_per_pixel = 10;  // 每个像素的采样数

for (int sample = 0; sample < samples_per_pixel; sample++) {
    ray r = get_ray(i, j);
    pixel_color += ray_color(r, world);
}
write_color(std::cout, pixel_samples_scale * pixel_color);
// pixel_samples_scale = 1.0 / samples_per_pixel
```

### 5.2 随机采样偏移

```cpp
vec3 sample_square() const {
    return vec3(random_double() - 0.5, random_double() - 0.5, 0);
}

ray get_ray(int i, int j) const {
    auto offset = sample_square();
    auto pixel_sample = pixel00_loc
                      + ((i + offset.x()) * pixel_delta_u)
                      + ((j + offset.y()) * pixel_delta_v);
    return ray(center, pixel_sample - center);
}
```

在 [-0.5, 0.5]² 范围内随机偏移像素采样位置，对边界进行模糊。

### 5.3 Gamma 校正与色域钳制

```cpp
static const interval intensity(0.000, 0.999);
int rbyte = int(256 * intensity.clamp(r));
```

将颜色值钳制到 [0, 0.999]，避免浮点溢出导致输出异常。

---

## 六、漫反射材质

### 6.1 理想漫反射

- 不发光但吸收光线（颜色越暗吸收越多）
- 反射方向完全随机（在法线方向的半球内）

**算法：**
1. 光线击中表面
2. 随机选择一个半球内的反射方向
3. 递归追踪，颜色乘以 0.5（吸收一半）

### 6.2 半球随机方向生成

```cpp
vec3 random_on_hemisphere(const vec3& normal) {
    vec3 on_unit_sphere = random_unit_vector();
    if (dot(on_unit_sphere, normal) > 0.0)
        return on_unit_sphere;
    else
        return -on_unit_sphere;
}
```

### 6.3 递归深度限制

```cpp
int max_depth = 10;

color ray_color(const ray& r, int depth, const hittable& world) {
    if (depth <= 0) return color(0,0,0);  // 超过深度 → 黑色

    if (world.hit(r, interval(0.001, infinity), rec)) {
        vec3 direction = random_on_hemisphere(rec.normal);
        return 0.5 * ray_color(ray(rec.p, direction), depth-1, world);
    }
    // 未击中 → 背景渐变（天空色）
    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0-a)*color(1.0,1.0,1.0) + a*color(0.5,0.7,1.0);
}
```

### 6.4 Shadow Acne 修复

当射线击中表面时，由于浮点精度误差，交点可能略微在表面之下，导致射线"击中自身"。修正：将 `t_min` 设为 `0.001` 而非 `0`，忽略极近距离的交点。

```cpp
if (world.hit(r, interval(0.001, infinity), rec))  // 不是 interval(0, infinity)
```

---

## 七、材质系统

### 7.1 材质抽象类

```cpp
class material {
    virtual bool scatter(
        const ray& r_in, const hit_record& rec,
        color& attenuation, ray& scattered
    ) const = 0;
};
```

- 返回 `true`：光线被散射（反射/折射）
- `attenuation`：该次散射的颜色衰减（反照率）
- `scattered`：散射后的新光线

### 7.2 Lambertian 材质（理想漫反射）

```cpp
class lambertian : public material {
    bool scatter(const ray& r_in, const hit_record& rec,
                 color& attenuation, ray& scattered) const override {
        auto scatter_direction = rec.normal + random_unit_vector();

        // 防止散射线为零向量（与法线完全反向的情况）
        if (scatter_direction.near_zero())
            scatter_direction = rec.normal;

        scattered = ray(rec.p, scatter_direction);
        attenuation = albedo;
        return true;
    }
    color albedo;  // 反射率（物体颜色）
};
```

### 7.3 金属（Metal）材质

**镜面反射公式：**
```cpp
vec3 reflect(const vec3& v, const vec3& n) {
    return v - 2 * dot(v, n) * n;
}
```

反射方向 $\mathbf{r} = \mathbf{v} - 2(\mathbf{v} \cdot \mathbf{n})\mathbf{n}$。

**模糊反射（Fuzziness）：**

```cpp
class metal : public material {
    metal(const color& albedo, double fuzz) : albedo(albedo), fuzz(fuzz < 1 ? fuzz : 1) {}

    bool scatter(const ray& r_in, const hit_record& rec,
                 color& attenuation, ray& scattered) const override {
        vec3 reflected = reflect(r_in.direction(), rec.normal);
        reflected = unit_vector(reflected) + (fuzz * random_unit_vector());
        scattered = ray(rec.p, reflected);
        attenuation = albedo;
        // 防止模糊后散射到表面下方
        return (dot(scattered.direction(), rec.normal) > 0);
    }
    color albedo;
    double fuzz;  // 模糊球半径（0 = 镜面，越大越漫反射）
};
```

- 反射光线归一化后再加扰动
- 如果散射线在表面下方，吸收该光线

### 7.4 电介质（Dielectric/玻璃）材质

**Snell定律（折射定律）：**
$$
\eta \cdot \sin\theta = \eta' \cdot \sin\theta'
$$

**折射光线计算：**
```cpp
vec3 refract(const vec3& uv, const vec3& n, double etai_over_etat) {
    auto cos_theta = fmin(dot(-uv, n), 1.0);
    vec3 r_out_perp = etai_over_etat * (uv + cos_theta * n);
    vec3 r_out_parallel = -sqrt(fabs(1.0 - r_out_perp.length_squared())) * n;
    return r_out_perp + r_out_parallel;
}
```

将折射光线分解为垂直于法线的分量 $R'_{\perp}$ 和平行于法线的分量 $R'_{\parallel}$：

$$
R'_{\perp} = \frac{\eta}{\eta'}(R + \cos\theta \cdot \mathbf{n})
$$
$$
R'_{\parallel} = -\sqrt{1 - |R'_{\perp}|^2} \cdot \mathbf{n}
$$

**全内反射（Total Internal Reflection）：**

当 $\frac{\eta}{\eta'} \sin\theta > 1$ 时无法折射，所有光线被反射。

```cpp
double cos_theta = fmin(dot(-unit_direction, rec.normal), 1.0);
double sin_theta = sqrt(1.0 - cos_theta * cos_theta);
bool cannot_refract = ri * sin_theta > 1.0;

if (cannot_refract)
    direction = reflect(unit_direction, rec.normal);
else
    direction = refract(unit_direction, rec.normal, ri);
```

**折射率计算**：`rec.front_face ? (1.0/refraction_index) : refraction_index`

- 从外部射入：空气(1.0) → 介质(refraction_index)，比值为 1.0/refraction_index
- 从内部射出：介质 → 空气，比值为 refraction_index

### 7.5 Schlick 近似（菲涅尔效应）

真实玻璃在掠射角时有更高反射率。Schlick 多项式近似：

```cpp
double reflectance(cosine, refraction_index) {
    auto r0 = (1 - refraction_index) / (1 + refraction_index);
    r0 = r0 * r0;
    return r0 + (1 - r0) * pow((1 - cosine), 5);
}
```

### 7.6 空心玻璃球

内球折射率 = 空气折射率 / 外球折射率（如 1.00/1.50 = 0.67）

```cpp
auto material_outer = make_shared<dielectric>(1.50);
auto material_inner = make_shared<dielectric>(1.00 / 1.50);
```

---

## 八、可定位相机与景深

### 8.1 可调FOV相机

```cpp
auto theta = degrees_to_radians(vfov);
auto h = tan(theta / 2);
auto viewport_height = 2 * h * focal_length;
```

### 8.2 可定位/定向相机

使用 lookfrom、lookat 和 vup 三个参数：

- **w**：lookfrom - lookat 方向（与视野相反，-Z）
- **u**：vup × w 的方向（相机的右方向，X）
- **v**：w × u 的方向（相机的上方向，Y）

```cpp
w = unit_vector(lookfrom - lookat);
u = unit_vector(cross(vup, w));
v = cross(w, u);
```

相机中心移到 lookfrom，视口方向基于 (u, v, w) 正交基。

### 8.3 景深（Defocus Blur）

真实相机的光圈不是针孔，大光圈会产生散焦模糊（景深效果）。

**模拟方法**：将光线起点随机偏移到一个圆盘上（模拟镜头光圈），所有光线聚焦在焦平面上。

```cpp
// 焦平面上的交点
auto defocus_disk_u = u * defocus_radius;
auto defocus_disk_v = v * defocus_radius;

// 从光圈圆盘上的随机点发射光线
point3 defocus_disk_sample = center + defocus_disk_u * p.x() + defocus_disk_v * p.y();
ray_origin = defocus_disk_sample;
ray_direction = pixel_sample - ray_origin;  // 所有光线聚焦在焦平面上
```

---

## 九、ray_color 最终流程

```cpp
color ray_color(const ray& r, int depth, const hittable& world) {
    if (depth <= 0) return color(0,0,0);

    hit_record rec;
    if (world.hit(r, interval(0.001, infinity), rec)) {
        ray scattered;
        color attenuation;
        if (rec.mat->scatter(r, rec, attenuation, scattered))
            return attenuation * ray_color(scattered, depth-1, world);
        return color(0,0,0);
    }

    // 背景渐变
    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0-a)*color(1.0,1.0,1.0) + a*color(0.5,0.7,1.0);
}
```

核心循环：光线发出 → 击中物体 → 材质决定如何散射 → 递归计算 → 累乘衰减系数。最大深度用俄罗斯轮盘赌思想简化处理。
