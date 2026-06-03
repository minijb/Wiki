---
title: Ray Tracing in One Weekend 笔记
type: source
updated: 2026-06-02
tags:
  - raytracing
  - cpp
  - graphics
  - path-tracing
  - rendering
---

# Ray Tracing in One Weekend 笔记

> 基于 Peter Shirley 的系列教程，从零实现一个完整的光线追踪器（Path Tracer）。

---

## 一、基础框架

### 1.1 PPM 图像输出

```cpp
std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";
// 逐像素输出 RGB 值
```

P3 格式表示 ASCII 编码的彩色 PPM 图像。渲染时逐行输出像素颜色值，`std::clog` 输出扫描线进度。

### 1.2 vec3 类

三维向量类是光线追踪器的核心数据结构。同时用于表示颜色（`color = vec3`）和空间点（`point3 = vec3`），通过类型别名区分语义。

```cpp
class vec3 {
    double e[3];
    double x() const { return e[0]; }
    double y() const { return e[1]; }
    double z() const { return e[2]; }
    // 基本运算：+、-、*、/、+=、*=、/=
    double length() const;
    double length_squared() const;
    static vec3 random();         // [0,1) 随机向量
    static vec3 random(min, max); // [min,max) 随机向量
    bool near_zero() const;       // 接近零向量判断（阈值 1e-8）
};

// 自由函数
double dot(const vec3& u, const vec3& v);
vec3 cross(const vec3& u, const vec3& v);
vec3 unit_vector(const vec3& v);
vec3 random_in_unit_sphere();
vec3 random_unit_vector();
vec3 random_on_hemisphere(const vec3& normal);
vec3 random_in_unit_disk();
```

**random_in_unit_sphere**：用拒绝法在单位球内生成随机点。不断在 [-1,1]³ 中随机采样，接受长度 < 1 的点。

**random_unit_vector**：标准化 `random_in_unit_sphere` 的结果。

**random_on_hemisphere**：如果随机方向与法线夹角 > 90°（点积 < 0），则反向。

**random_in_unit_disk**：在单位圆盘上随机采样（用于景深效果），用拒绝法在 [-1,1]² 中采样。

### 1.3 随机数工具

```cpp
inline double random_double() {
    return rand() / (RAND_MAX + 1.0);  // 返回 [0,1)
}

inline double random_double(double min, double max) {
    return min + (max-min) * random_double();  // 返回 [min,max)
}
```

### 1.4 Interval 类（区间管理）

用于管理光线参数的合法范围（t_min, t_max）：

```cpp
class interval {
    double min, max;
    bool contains(double x) const { return min <= x && x <= max; }
    bool surrounds(double x) const { return min < x && x < max; }
    static const interval empty;     // (+∞, -∞)
    static const interval universe;  // (-∞, +∞)
};
```

### 1.5 Gamma 校正

显示器通常在 Gamma 2 空间工作。将线性颜色转换到 Gamma 空间使亮度过渡更平滑：

```cpp
inline double linear_to_gamma(double linear_component) {
    if (linear_component > 0)
        return sqrt(linear_component);  // γ=2，故取平方根
    return 0;
}

void write_color(std::ostream& out, const color& pixel_color) {
    auto r = linear_to_gamma(pixel_color.x());
    auto g = linear_to_gamma(pixel_color.y());
    auto b = linear_to_gamma(pixel_color.z());
    static const interval intensity(0.000, 0.999);
    int rbyte = int(256 * intensity.clamp(r));
    int gbyte = int(256 * intensity.clamp(g));
    int bbyte = int(256 * intensity.clamp(b));
    out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}
```

> [!note] 为什么需要 Gamma 校正
> 人眼对暗部变化更敏感。线性颜色存储后，暗部会丢失细节。Gamma 校正重新分配位深度，使暗部获得更多精度。

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

### 2.2 基础相机与视口

> [!info] 光线是反向的
> 光线追踪从相机发射光线到场景——与真实光线传播方向相反（光路可逆原理）。

**关键参数：**
- 宽高比：`aspect_ratio = 16.0 / 9.0`
- 视口高度：`viewport_height = 2.0`（3D 世界中的任意值）
- 焦距：`focal_length = 1.0`（相机中心到视口平面的距离）
- 相机中心：原点 `(0, 0, 0)`

**像素网格：**
- 像素 (i, j) 从左上角开始，逐行向下扫描
- 图像坐标 Y 轴向下（与 3D 空间 Y 轴向上相反）
- 像素网格从视口边缘内缩半个像素间距，使得视口区域均匀划分

**光线发射：**
```cpp
ray get_ray(int i, int j) const {
    auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
    auto ray_direction = pixel_center - center;
    return ray(center, ray_direction);
}
```

### 2.3 Camera 类封装

相机类有两个公共方法（`initialize`、`render`）和两个私有方法（`get_ray`、`ray_color`）：

```cpp
class camera {
  public:
    double aspect_ratio = 1.0;
    int    image_width  = 100;
    void render(const hittable& world);

  private:
    int    image_height;
    point3 center;
    point3 pixel00_loc;
    vec3   pixel_delta_u;
    vec3   pixel_delta_v;
    void initialize();
    ray get_ray(int i, int j) const;
    color ray_color(const ray& r, const hittable& world) const;
};
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

> [!tip] 为什么简化 b = -2h
> $-b \pm \sqrt{b^2-4ac} \over 2a$ → 代入 b=-2h → $h \pm \sqrt{h^2-ac} \over a$，消去除以 2 的操作。

### 3.3 法线计算

球体上一点 P 的外向法线：$\mathbf{n} = (P - C) / r$（单位向量）。

### 3.4 渲染法线颜色

将法线的 [-1,1] 范围映射到 [0,1] 颜色：
```cpp
rec.normal = (rec.p - center) / radius;
// 颜色 = 0.5 * (normal + 1)
```

---

## 四、Hittable 抽象与正面判断

### 4.1 Hittable 抽象类

```cpp
class hit_record {
    point3 p;            // 交点
    vec3 normal;         // 法线（始终指向光线来向）
    shared_ptr<material> mat;  // 材质指针
    double t;            // 光线参数
    bool front_face;     // 是否在物体外表面
    void set_face_normal(const ray& r, const vec3& outward_normal);
};

class hittable {
    virtual bool hit(const ray& r, interval ray_t, hit_record& rec) const = 0;
};
```

> [!note] 为什么需要 front_face
> 对于玻璃等内外两面的物体，需要知道光线是从外部射入还是内部射出，以确定折射率比值。

### 4.2 正面/背面判断

```cpp
void set_face_normal(const ray& r, const vec3& outward_normal) {
    front_face = dot(r.direction(), outward_normal) < 0;
    normal = front_face ? outward_normal : -outward_normal;
}
```

- `front_face = true`：光线从外部射入（法线与光线反向）
- `front_face = false`：光线从内部射出（法线反转）

有两种策略：法线总是指向外（需要存 front_face），或法线总是指向光线。本书选择前者。

### 4.3 球体实现

```cpp
class sphere : public hittable {
    bool hit(const ray& r, interval ray_t, hit_record& rec) const {
        // 求解二次方程
        // 用 ray_t.surrounds(root) 在 [t_min, t_max] 范围内找最近的根
        rec.t = root;
        rec.p = r.at(rec.t);
        vec3 outward_normal = (rec.p - center) / radius;
        rec.set_face_normal(r, outward_normal);
        rec.mat = mat;
        return true;
    }
};
```

### 4.4 Hittable List

场景是多个可击中物体的集合：

```cpp
class hittable_list : public hittable {
    vector<shared_ptr<hittable>> objects;

    bool hit(const ray& r, interval ray_t, hit_record& rec) const {
        hit_record temp_rec;
        bool hit_anything = false;
        auto closest_so_far = ray_t.max;

        for (const auto& object : objects) {
            if (object->hit(r, interval(ray_t.min, closest_so_far), temp_rec)) {
                hit_anything = true;
                closest_so_far = temp_rec.t;  // 收缩搜索范围
                rec = temp_rec;
            }
        }
        return hit_anything;
    }
};
```

遍历所有物体，每次找到更近的交点就收缩 `closest_so_far`。

---

## 五、反走样（Antialiasing）

### 5.1 多重采样

对每个像素发射多条随机偏移的光线，取颜色平均值：

```cpp
int samples_per_pixel = 10;  // 每个像素的采样数

for (int j = 0; j < image_height; j++) {
    for (int i = 0; i < image_width; i++) {
        color pixel_color(0,0,0);
        for (int sample = 0; sample < samples_per_pixel; sample++) {
            ray r = get_ray(i, j);
            pixel_color += ray_color(r, world);
        }
        write_color(std::cout, pixel_samples_scale * pixel_color);
        // pixel_samples_scale = 1.0 / samples_per_pixel
    }
}
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

在 [-0.5, 0.5]² 范围内随机偏移像素采样位置，采样以像素为中心延伸到四邻像素一半的正方形区域。对边界进行模糊，消除锯齿。

---

## 六、漫反射材质

### 6.1 理想漫反射

- 不发光但吸收光线（颜色越暗吸收越多）
- 反射方向完全随机（在法线方向的半球内）

**算法：**
1. 光线击中表面
2. 随机选择一个半球内的反射方向
3. 递归追踪，颜色乘以衰减系数（如 0.5）

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

### 6.3 递归深度限制与 Shadow Acne

```cpp
int max_depth = 10;

color ray_color(const ray& r, int depth, const hittable& world) {
    if (depth <= 0) return color(0,0,0);  // 超过深度 → 黑色（吸收所有光）

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

### 6.5 True Lambertian Reflection

更真实的漫反射：散射方向的分布与 $\cos\theta$ 成正比（更多光线沿法线方向散射）。

实现方式：在交点处有一个以 $P + \mathbf{n}$ 为球心（表面外侧）的单位球，在其表面随机取点 S，则散射方向为 $S - P$：

```cpp
// Lambertian scatter: 法线 + 随机单位向量
vec3 scatter_direction = rec.normal + random_unit_vector();
// 如果恰好为零向量（与法线完全反向），回退到法线方向
if (scatter_direction.near_zero())
    scatter_direction = rec.normal;
```

这等价于在切线球上均匀采样，产生的分布满足 $\cos\theta$ 规律，比半球均匀采样更真实。

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

> [!note] 循环依赖解决
> `hit_record` 需要 `material*`，`material::scatter` 需要 `hit_record`。在 C++ 中使用前向声明 `class material;` 解决，因为只存指针。

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

反射方向 $\mathbf{r} = \mathbf{v} - 2(\mathbf{v} \cdot \mathbf{n})\mathbf{n}$。推导：v 在 n 上的投影长度为 $\mathbf{v} \cdot \mathbf{n}$（负方向指向表面），反射需要减去两倍投影。

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
        // 防止模糊后散射到表面下方 → 吸收该光线
        return (dot(scattered.direction(), rec.normal) > 0);
    }
    color albedo;
    double fuzz;  // 模糊球半径（0 = 镜面，越大越漫反射）
};
```

- 反射光线归一化后再加扰动
- 如果散射线在表面下方（点积 < 0），吸收该光线
- `fuzz` 钳制到 1 以下

### 7.4 电介质（Dielectric/玻璃）材质

**Snell 定律（折射定律）：**
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

> [!info] 全内反射现象
> 这就是为什么在水下向上看时，水面像一面镜子。光线从水（折射率 1.33）到空气（1.00），掠射角足够大时发生全内反射。

**折射率计算**：`rec.front_face ? (1.0/refraction_index) : refraction_index`

- 从外部射入：空气(1.0) → 介质(refraction_index)，比值为 1.0/refraction_index
- 从内部射出：介质 → 空气，比值为 refraction_index

### 7.5 Schlick 近似（菲涅尔效应）

真实玻璃在掠射角时有更高反射率。Schlick 多项式近似：

```cpp
double reflectance(double cosine, double refraction_index) {
    auto r0 = (1 - refraction_index) / (1 + refraction_index);
    r0 = r0 * r0;
    return r0 + (1 - r0) * pow((1 - cosine), 5);
}
```

通过随机数与 `reflectance` 比较，决定散射还是折射。Attenuation 始终为 1（玻璃表面不吸收光）。

### 7.6 空心玻璃球

内球折射率 = 空气折射率 / 外球折射率（如 1.00/1.50 = 0.67）：

```cpp
auto material_outer = make_shared<dielectric>(1.50);
auto material_inner = make_shared<dielectric>(1.00 / 1.50);
world.add(make_shared<sphere>(center, 0.5, material_outer));
world.add(make_shared<sphere>(center, 0.4, material_inner));  // 略小于外球
```

光线路径：击中外球 → 折射进入 → 击中内球 → 折射进入空气 → 继续追踪...

---

## 八、可定位相机与景深

### 8.1 可调FOV相机

```cpp
auto theta = degrees_to_radians(vfov);
auto h = tan(theta / 2);
auto viewport_height = 2 * h * focus_dist;
```

FOV 与视口的关系：$\tan(vfov/2) = viewport\_height / (2 \cdot focus\_dist)$

### 8.2 可定位/定向相机

使用 lookfrom、lookat 和 vup 三个参数构建正交基 (u, v, w)：

- **w**：`unit_vector(lookfrom - lookat)` — 与视野方向相反（-Z）
- **u**：`unit_vector(cross(vup, w))` — 相机的右方向（X）
- **v**：`cross(w, u)` — 相机的上方向（Y）

```cpp
w = unit_vector(lookfrom - lookat);
u = unit_vector(cross(vup, w));
v = cross(w, u);
```

> [!tip] vup 只要不平行于视线即可
> vup 投影到与 w 正交的平面上 → 叉乘 → 完整的正交基。典型值 `(0, 1, 0)`。

视口方向和像素增量基于 (u, v, w)：
```cpp
vec3 viewport_u = viewport_width * u;     // 水平方向
vec3 viewport_v = viewport_height * -v;   // 垂直方向（向下）
```

相机中心移到 lookfrom，视口位于 `center - focus_dist * w`。

### 8.3 景深（Defocus Blur）

真实相机的光圈不是针孔，大光圈会产生散焦模糊（景深效果）。

**薄透镜近似**：从镜头圆盘上的随机点发射光线，所有光线聚焦在焦平面上。

```cpp
// 镜头圆盘半径
auto defocus_radius = focus_dist * tan(degrees_to_radians(defocus_angle / 2));
defocus_disk_u = u * defocus_radius;
defocus_disk_v = v * defocus_radius;

// 从光圈圆盘上的随机点发射光线
point3 defocus_disk_sample = center + p.x() * defocus_disk_u + p.y() * defocus_disk_v;
ray_origin = (defocus_angle <= 0) ? center : defocus_disk_sample;
ray_direction = pixel_sample - ray_origin;  // 所有光线聚焦在焦平面上（pixel_sample 在焦平面上）
```

**关键参数：**
- `defocus_angle`：镜头锥角，控制模糊程度（0 = 针孔，无模糊）
- `focus_dist`：相机中心到完美聚焦平面的距离
- 焦平面上的物体完美聚焦，距离越远越模糊

焦点距离和焦距的区别：在薄透镜近似中，将像素网格放在焦平面上，两者相等。

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

---

## 十、完整场景示例

```cpp
int main() {
    hittable_list world;

    auto ground_material = make_shared<lambertian>(color(0.5, 0.5, 0.5));
    world.add(make_shared<sphere>(point3(0, -1000, 0), 1000, ground_material));

    // 随机小球的场景
    for (int a = -11; a < 11; a++) {
        for (int b = -11; b < 11; b++) {
            auto choose_mat = random_double();
            point3 center(a + 0.9*random_double(), 0.2, b + 0.9*random_double());
            if ((center - point3(4, 0.2, 0)).length() > 0.9) {
                if (choose_mat < 0.8) {
                    // 漫反射
                    auto albedo = color::random() * color::random();
                    world.add(make_shared<sphere>(center, 0.2,
                        make_shared<lambertian>(albedo)));
                } else if (choose_mat < 0.95) {
                    // 金属
                    auto albedo = color::random(0.5, 1);
                    auto fuzz = random_double(0, 0.5);
                    world.add(make_shared<sphere>(center, 0.2,
                        make_shared<metal>(albedo, fuzz)));
                } else {
                    // 玻璃
                    world.add(make_shared<sphere>(center, 0.2,
                        make_shared<dielectric>(1.5)));
                }
            }
        }
    }

    // 三个大球
    world.add(make_shared<sphere>(point3( 0, 1, 0), 1.0, make_shared<dielectric>(1.5)));
    world.add(make_shared<sphere>(point3(-4, 1, 0), 1.0, make_shared<lambertian>(color(0.4, 0.2, 0.1))));
    world.add(make_shared<sphere>(point3( 4, 1, 0), 1.0, make_shared<metal>(color(0.7, 0.6, 0.5), 0.0)));

    camera cam;
    cam.aspect_ratio      = 16.0 / 9.0;
    cam.image_width       = 1200;
    cam.samples_per_pixel = 500;
    cam.max_depth         = 50;
    cam.vfov              = 20;
    cam.lookfrom          = point3(13,2,3);
    cam.lookat            = point3(0,0,0);
    cam.vup               = vec3(0,1,0);
    cam.defocus_angle     = 0.6;
    cam.focus_dist        = 10.0;
    cam.render(world);
}
```

---

## 十一、教程后续：多线程与性能

原教程后续涉及 **The Next Week** 和 **The Rest of Your Life** 两本续作：
- **The Next Week**：运动模糊（Motion Blur）、BVH 加速结构、纹理（固体纹理/图像纹理）、Perlin 噪声、矩形与光源、体积渲染、最终场景
- **The Rest of Your Life**：蒙特卡洛理论基础、重要性采样、混合密度、分层采样

**多线程渲染**：由于每个像素的光线追踪完全独立，可以通过 OpenMP 或线程池并行化像素循环：
```cpp
#pragma omp parallel for schedule(dynamic)
for (int j = 0; j < image_height; j++) {
    for (int i = 0; i < image_width; i++) {
        // 渲染像素 (i, j)
    }
}
```

---

## 相关页面

- [[光线追踪入门]] — 概念页面
- [[渲染管线理论]] — GAMES101/202 光线追踪与路径追踪理论
- [[Shader高级特性]] — Unity 阴影系统
- [[ray-tracing-in-one-weekend-摘要]] — 精简摘要
