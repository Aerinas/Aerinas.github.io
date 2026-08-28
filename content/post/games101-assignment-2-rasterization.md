+++
title = 'GAMES101 Assignment 2：三角形光栅化与深度缓冲'
date = 2026-08-28T15:35:19+10:00
draft = false
description = 'GAMES101 Assignment 2 学习笔记：包围盒、像素中心采样、三角形内测试、重心坐标、透视正确插值、Z-buffer 与 MSAA。'
categories = ['计算机图形学']
tags = ['GAMES101', 'Assignment 2', '光栅化', '重心坐标', 'Z-buffer', 'MSAA', 'C++']
series = ['GAMES101 作业笔记']
url = '/post/games101-assignment-2/'
+++

Assignment 1 把三维顶点变换到了屏幕空间，但屏幕上仍然只有三个顶点。Assignment 2 要继续回答一个更具体的问题：一个三角形究竟覆盖了哪些像素，被多个三角形覆盖时又该显示哪一个？

这篇笔记按软件光栅器的实际执行顺序，整理三角形覆盖测试、属性插值、深度测试与抗锯齿。由于当前博客目录里没有保存 Assignment 2 的原始作业代码，文中的 C++17 代码使用与课程框架兼容的通用写法，接口名可能需要按本地框架微调。

## 作业目标

基础部分需要完成：

1. 求三角形在屏幕上的二维包围盒；
2. 遍历包围盒内的像素中心；
3. 判断采样点是否位于三角形内部；
4. 使用重心坐标插值深度；
5. 使用 Z-buffer 解决遮挡关系；
6. 将通过深度测试的颜色写入帧缓冲。

提高部分通常是实现 2×2 超采样，让三角形边缘变得平滑。

整条流程可以概括为：

```text
屏幕空间三角形
      ↓
计算二维包围盒
      ↓
对候选像素/子采样点做覆盖测试
      ↓
计算重心坐标并插值深度
      ↓
执行深度测试
      ↓
更新深度缓冲和颜色缓冲
```

## 一、为什么选择三角形

三角形是光栅化中最常用的基本图元，主要因为：

- 任意三个不共线的点都能确定一个平面；
- 三角形内部有明确的定义；
- 复杂多边形可以拆成三角形；
- 顶点属性可以用重心坐标稳定地插值到内部。

光栅化的工作不是把连续平面“完整地”塞进像素，而是在有限的采样位置上判断三角形是否覆盖了它。默认每个像素只取一个样本时，通常检测像素中心：

```cpp
const float sampleX = static_cast<float>(x) + 0.5f;
const float sampleY = static_cast<float>(y) + 0.5f;
```

这里的 `x`、`y` 是像素索引，而 `x + 0.5`、`y + 0.5` 才是像素中心的屏幕坐标。混用两者会让图形产生半个像素的偏移。

## 二、用包围盒缩小搜索范围

最直接的做法是遍历整张屏幕，但大多数像素都不在当前三角形附近。更合理的方法是先计算三角形的轴对齐包围盒（AABB）：

```cpp
const float minXf = std::min({v[0].x(), v[1].x(), v[2].x()});
const float maxXf = std::max({v[0].x(), v[1].x(), v[2].x()});
const float minYf = std::min({v[0].y(), v[1].y(), v[2].y()});
const float maxYf = std::max({v[0].y(), v[1].y(), v[2].y()});

const int minX = std::max(0, static_cast<int>(std::floor(minXf)));
const int maxX = std::min(width - 1, static_cast<int>(std::ceil(maxXf)));
const int minY = std::max(0, static_cast<int>(std::floor(minYf)));
const int maxY = std::min(height - 1, static_cast<int>(std::ceil(maxYf)));
```

`floor` 保证左下边界不会漏掉候选像素，`ceil` 保证右上边界不会过早收缩。最后还要裁剪到帧缓冲的合法范围，否则三角形部分落在屏幕外时会发生数组越界。

## 三、判断点是否在三角形内

设三角形顶点为 `A`、`B`、`C`，待检测点为 `P`。对于有固定绕序的三角形，如果 `P` 位于内部，那么它相对于三条有向边应当始终位于同一侧。

二维叉积只需要计算 Z 分量：

\[
\operatorname{cross}(\mathbf a,\mathbf b)
=a_xb_y-a_yb_x
\]

分别计算：

\[
(B-A)\times(P-A)
\]

\[
(C-B)\times(P-B)
\]

\[
(A-C)\times(P-C)
\]

三个结果同号，点就在三角形内部或边界上。

```cpp
static float edgeFunction(const Eigen::Vector4f& a,
                          const Eigen::Vector4f& b,
                          float x,
                          float y)
{
    return (b.x() - a.x()) * (y - a.y())
         - (b.y() - a.y()) * (x - a.x());
}

static bool insideTriangle(float x,
                           float y,
                           const Eigen::Vector4f* v)
{
    const float e0 = edgeFunction(v[0], v[1], x, y);
    const float e1 = edgeFunction(v[1], v[2], x, y);
    const float e2 = edgeFunction(v[2], v[0], x, y);

    const bool hasNegative = e0 < 0.0f || e1 < 0.0f || e2 < 0.0f;
    const bool hasPositive = e0 > 0.0f || e1 > 0.0f || e2 > 0.0f;
    return !(hasNegative && hasPositive);
}
```

同时接受“全为非负”和“全为非正”，就不会依赖顶点是顺时针还是逆时针排列。对于退化成线段或点的三角形，应在上游通过面积阈值排除。

### 边界归属

两个相邻三角形共享一条边时，如果双方都把边界算作内部，可能重复绘制；如果双方都排除，又会产生裂缝。实际图形 API 常使用 top-left rule，为共享边规定唯一归属。课程作业的简单场景通常用同号判断就足够，但构建更完整的光栅器时应该补上统一的边界规则。

## 四、重心坐标

点 `P` 位于三角形内部时，可以写成三个顶点的加权和：

\[
P=\alpha A+\beta B+\gamma C
\]

并且：

\[
\alpha+\beta+\gamma=1
\]

三角形内部的三个权重均为非负。靠近哪个顶点，对应权重就越大；落在某条边上时，对面顶点的权重为零。

课程框架常见的计算函数如下：

```cpp
static std::tuple<float, float, float>
computeBarycentric2D(float x,
                     float y,
                     const Eigen::Vector4f* v)
{
    const float alpha =
        (x * (v[1].y() - v[2].y())
       + (v[2].x() - v[1].x()) * y
       + v[1].x() * v[2].y()
       - v[2].x() * v[1].y())
        /
        (v[0].x() * (v[1].y() - v[2].y())
       + (v[2].x() - v[1].x()) * v[0].y()
       + v[1].x() * v[2].y()
       - v[2].x() * v[1].y());

    const float beta =
        (x * (v[2].y() - v[0].y())
       + (v[0].x() - v[2].x()) * y
       + v[2].x() * v[0].y()
       - v[0].x() * v[2].y())
        /
        (v[1].x() * (v[2].y() - v[0].y())
       + (v[0].x() - v[2].x()) * v[1].y()
       + v[2].x() * v[0].y()
       - v[0].x() * v[2].y());

    return {alpha, beta, 1.0f - alpha - beta};
}
```

重心坐标不仅可以插值深度，后续还会用于插值颜色、法线、纹理坐标和观察空间位置，是整个光栅化管线中非常通用的工具。

## 五、为什么需要透视正确插值

屏幕空间经历过透视除法，直接对顶点属性做线性插值，会把原本在三维空间中线性变化的量插错。正确做法是先用每个顶点的 `1/w` 修正权重。

对于顶点属性 `q0`、`q1`、`q2`：

\[
q(P)=
\frac{
\alpha q_0/w_0+\beta q_1/w_1+\gamma q_2/w_2
}{
\alpha/w_0+\beta/w_1+\gamma/w_2
}
\]

深度插值可以写成：

```cpp
const auto [alpha, beta, gamma] = computeBarycentric2D(x, y, v);

const float reciprocalW =
    1.0f / (alpha / v[0].w()
          + beta  / v[1].w()
          + gamma / v[2].w());

const float depth =
    (alpha * v[0].z() / v[0].w()
   + beta  * v[1].z() / v[1].w()
   + gamma * v[2].z() / v[2].w()) * reciprocalW;
```

Assignment 2 的框架可能已经处理了部分插值细节，但理解这条公式很重要。到纹理映射阶段，错误的线性插值会产生明显的纹理扭曲。

## 六、Z-buffer 解决遮挡

画家算法要求按从远到近排序物体，三角形互相穿插时很难处理。Z-buffer 则为每个像素记录当前见过的最近深度，使绘制顺序不再重要。

如果约定深度值越小越接近相机，初始化和测试逻辑为：

```cpp
std::fill(depthBuffer.begin(),
          depthBuffer.end(),
          std::numeric_limits<float>::infinity());

if (depth < depthBuffer[index]) {
    depthBuffer[index] = depth;
    frameBuffer[index] = color;
}
```

判断方向必须服从当前投影和视口映射的深度约定。不要仅凭“相机朝负 Z”猜测比较符号，而应打印近、远两个测试点经过全部变换后的实际深度值。

深度测试通过时，颜色缓冲和深度缓冲必须一起更新；只更新其中一个会造成后续三角形与当前画面不一致。

## 七、完整光栅化流程

下面的代码展示核心结构。它不是课程框架文件的逐字副本，其中的 `width`、`height`、`getIndex` 和 `setPixel` 需要对应到实际项目接口。

```cpp
void Rasterizer::rasterizeTriangle(const Triangle& triangle)
{
    const auto v = triangle.toVector4();

    const int minX = std::max(
        0,
        static_cast<int>(std::floor(
            std::min({v[0].x(), v[1].x(), v[2].x()}))));
    const int maxX = std::min(
        width - 1,
        static_cast<int>(std::ceil(
            std::max({v[0].x(), v[1].x(), v[2].x()}))));
    const int minY = std::max(
        0,
        static_cast<int>(std::floor(
            std::min({v[0].y(), v[1].y(), v[2].y()}))));
    const int maxY = std::min(
        height - 1,
        static_cast<int>(std::ceil(
            std::max({v[0].y(), v[1].y(), v[2].y()}))));

    for (int y = minY; y <= maxY; ++y) {
        for (int x = minX; x <= maxX; ++x) {
            const float sampleX = static_cast<float>(x) + 0.5f;
            const float sampleY = static_cast<float>(y) + 0.5f;

            if (!insideTriangle(sampleX, sampleY, v.data())) {
                continue;
            }

            const auto [alpha, beta, gamma] =
                computeBarycentric2D(sampleX, sampleY, v.data());

            const float reciprocalW =
                1.0f / (alpha / v[0].w()
                      + beta  / v[1].w()
                      + gamma / v[2].w());

            const float depth =
                (alpha * v[0].z() / v[0].w()
               + beta  * v[1].z() / v[1].w()
               + gamma * v[2].z() / v[2].w()) * reciprocalW;

            const int index = getIndex(x, y);
            if (depth >= depthBuffer[index]) {
                continue;
            }

            depthBuffer[index] = depth;
            setPixel(x, y, triangle.getColor());
        }
    }
}
```

若 `toVector4()` 返回的是普通数组而不是 `std::array`，就不能调用 `v.data()`，应直接传 `v`。这正是把通用思路接入课程框架时需要调整的接口差异。

## 八、2×2 超采样与 MSAA

每个像素只检测中心一次时，边缘只能做“覆盖”或“不覆盖”的二选一判断，于是斜边呈现台阶。2×2 超采样把每个像素拆成四个子采样点，例如：

```cpp
constexpr std::array<Eigen::Vector2f, 4> offsets{{
    {0.25f, 0.25f},
    {0.75f, 0.25f},
    {0.25f, 0.75f},
    {0.75f, 0.75f}
}};
```

每个子采样点分别执行覆盖测试和深度测试，最后对四个颜色求平均：

\[
C_{pixel}=\frac{C_0+C_1+C_2+C_3}{4}
\]

只统计“四个点中有几个在三角形内”，然后用覆盖率乘颜色，在单个三角形演示中看起来可能正确；但当多个三角形在同一像素内互相遮挡时会失败。正确实现需要保存每个子样本自己的深度和颜色：

```cpp
const int sampleIndex = (y * width + x) * 4 + sampleId;

if (depth < sampleDepthBuffer[sampleIndex]) {
    sampleDepthBuffer[sampleIndex] = depth;
    sampleColorBuffer[sampleIndex] = triangle.getColor();
}
```

所有三角形处理完成后，再把四个子样本解析成一个像素颜色。

严格来说，SSAA 会对每个子样本执行完整着色，而硬件 MSAA 通常让多个子样本共享一次像素着色结果，但仍分别保存覆盖率和深度。Assignment 2 的提高题常被称为 MSAA，实际实现可能更接近简化的超采样；重点是理解“增加采样率，再重建像素颜色”。

## 九、容易出错的地方

### 1. 忘记使用像素中心

覆盖测试应该使用 `x + 0.5f`、`y + 0.5f`，而不是整数像素索引。

### 2. 包围盒没有裁剪

三角形超出屏幕时，负索引或超过 `width - 1`、`height - 1` 的索引会破坏内存。

### 3. 只接受一种顶点绕序

只判断三个叉积都大于零，会让另一种绕序的三角形完全消失。应接受同为非负或同为非正。

### 4. 深度比较方向写反

先验证变换后近处和远处的深度大小，再决定使用 `<` 还是 `>`。同时检查深度缓冲的初始值是否匹配。

### 5. 帧缓冲 Y 轴翻转越界

如果图像内存以左上角为原点，而光栅空间以左下角为原点，常见映射为：

```cpp
const int index = (height - 1 - y) * width + x;
```

注意中间的 `-1`。

### 6. 用线性插值替代透视正确插值

深度或纹理坐标在屏幕空间直接使用 `alpha * q0 + beta * q1 + gamma * q2`，可能产生随透视变化的错误。

### 7. MSAA 仍只保存每像素一个深度

覆盖同一像素但深度不同的多个图元需要逐子样本判断。每像素单深度无法表达这种情况。

### 8. 浮点边界不稳定

恰好落在边上的结果会受浮点误差影响。简单作业可使用小的 `epsilon`，完整实现则应统一采用 edge function 和 top-left rule。

## 十、推荐的调试顺序

不要一次写完全部流程再看最终图片。按下面的顺序验证，错误会更容易定位：

1. 只画一个三角形的包围盒，确认范围和屏幕边界正确；
2. 用黑白颜色显示 `insideTriangle` 的结果；
3. 将 `alpha`、`beta`、`gamma` 映射到 RGB，检查重心坐标是否连续；
4. 把深度值显示成灰度图，检查近远变化；
5. 加入第二个相交三角形，验证绘制顺序改变后结果不变；
6. 最后加入 2×2 子采样，并分别观察四个样本的覆盖情况。

## 十一、这次作业真正学到的内容

Assignment 1 解决“顶点在哪里”，Assignment 2 解决“哪些样本属于图元，以及哪个图元可见”。两次作业连起来，就形成了软件光栅器最小但完整的骨架：

```text
顶点数据
  → MVP 变换
  → 透视除法
  → 视口变换
  → 三角形覆盖测试
  → 属性插值
  → 深度测试
  → 写入帧缓冲
```

可以复用的核心模式是：

- 用 AABB 减少候选范围；
- 用 edge function 判断半平面和覆盖关系；
- 用重心坐标插值三角形内部属性；
- 用 `1/w` 做透视正确插值；
- 用 depth buffer 把可见性问题变成局部比较；
- 用多重采样缓解离散采样造成的几何锯齿。

掌握这些步骤后，后续的纹理、法线、Blinn–Phong 着色虽然会增加属性和计算量，但仍然运行在同一条光栅化管线上。
