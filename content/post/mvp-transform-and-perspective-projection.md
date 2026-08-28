+++
title = 'GAMES101 Assignment 1：MVP 变换与透视投影'
date = 2026-08-24T01:43:18+10:00
lastmod = 2026-08-28T15:35:19+10:00
draft = false
description = 'GAMES101 Assignment 1 学习笔记：从坐标空间、齐次坐标和相似三角形出发，推导模型、观察与透视投影矩阵。'
categories = ['计算机图形学']
tags = ['GAMES101', 'Assignment 1', 'MVP', '透视投影', 'OpenGL', 'Eigen', '渲染管线']
series = ['GAMES101 作业笔记']
url = '/post/games101-assignment-1/'
aliases = ['/post/mvp-transform-and-perspective-projection/']
+++

这篇文章整理的是 GAMES101 Assignment 1：Rotation and Projection。作业的重点不只是补齐几个矩阵，而是亲手打通“局部坐标如何变成屏幕像素”这条最基础的渲染路径。

## 作业目标与完成内容

Assignment 1 主要完成以下任务：

1. 构造绕 Z 轴旋转的模型矩阵；
2. 根据相机位置构造观察矩阵；
3. 根据视场角、宽高比和近远裁剪面构造透视投影矩阵；
4. 按照 `Projection * View * Model * position` 的顺序变换顶点；
5. 理解透视除法和视口变换如何把顶点送到屏幕上；
6. 可选地用 Rodrigues 公式实现绕任意轴旋转。

我从这次作业中得到的关键认识是：矩阵的每一行并不是魔法常量，它们分别对应坐标系约定、相机逆变换、视锥体压缩和深度区间映射。先固定约定，再推公式，比直接背矩阵可靠得多。

> 本文使用列向量、右手坐标系、相机朝负 Z 轴观察，并采用 OpenGL 风格的 NDC 深度范围。阅读其他资料或图形 API 的公式时，应先检查这些约定是否一致。

在三维图形学中，模型通常定义在自己的局部坐标系里，而屏幕最终只能显示二维像素。为了把一个三维顶点变换到屏幕上，需要依次经过模型变换、观察变换和投影变换，也就是常说的 MVP 变换：

\[
\mathbf p_{clip}
=
\mathbf P\mathbf V\mathbf M\mathbf p_{local}
\]

其中：

- \(M\)：Model Matrix，模型矩阵；
- \(V\)：View Matrix，观察矩阵；
- \(P\)：Projection Matrix，投影矩阵；
- \(\mathbf p_{local}\)：模型局部坐标；
- \(\mathbf p_{clip}\)：裁剪空间坐标。

本文采用以下约定：

- 使用列向量；
- 矩阵从右向左作用；
- 使用右手坐标系；
- 相机位于原点并朝负 Z 轴观察；
- 使用 OpenGL 风格的 NDC，深度范围为 \([-1,1]\)。

不同图形 API 对深度范围、Y 轴方向和矩阵存储方式的约定可能不同，因此实际使用时要先确认坐标系约定。

---

## 一、图形渲染中的坐标空间

一个顶点从模型到屏幕，通常会经历以下空间：

```text
模型局部坐标 Local Space
        ↓ Model Matrix
世界坐标 World Space
        ↓ View Matrix
观察坐标 View/Camera Space
        ↓ Projection Matrix
裁剪坐标 Clip Space
        ↓ Perspective Divide
标准化设备坐标 NDC
        ↓ Viewport Transform
屏幕坐标 Screen Space
```

每一步都解决一个不同的问题。

### 1. 模型空间

模型空间是物体自己的局部坐标系。

例如，一个三角形可以定义为：

```cpp
std::vector<Eigen::Vector3f> positions{
    { 2, 0, -2 },
    { 0, 2, -2 },
    {-2, 0, -2 }
};
```

这些坐标描述顶点相对于物体自身原点的位置，并不直接代表它在整个场景中的最终位置。

### 2. 世界空间

模型矩阵将物体从局部坐标系放置到世界坐标系中，包括：

- 缩放；
- 旋转；
- 平移。

### 3. 观察空间

观察矩阵把世界变换到以相机为中心的坐标系中。

在观察空间里：

- 相机位于原点；
- 相机朝固定方向观察；
- 所有物体的位置都相对于相机表示。

### 4. 裁剪空间

投影矩阵把相机的可见视锥体变换到便于裁剪的空间中。

此时坐标还是四维齐次坐标：

\[
(x_{clip},y_{clip},z_{clip},w_{clip})
\]

### 5. NDC

对裁剪坐标执行透视除法：

\[
x_{ndc}=\frac{x_{clip}}{w_{clip}}
\]

\[
y_{ndc}=\frac{y_{clip}}{w_{clip}}
\]

\[
z_{ndc}=\frac{z_{clip}}{w_{clip}}
\]

得到标准化设备坐标 NDC。可见区域通常被规范化到：

\[
x,y,z\in[-1,1]
\]

### 6. 屏幕空间

最后通过视口变换把 NDC 映射到具体的屏幕分辨率，例如 \(700\times700\)。

---

## 二、为什么使用齐次坐标

三维点原本表示为：

\[
\mathbf p=
\begin{bmatrix}
x\\y\\z
\end{bmatrix}
\]

为了统一描述旋转、缩放、平移和投影，需要将其扩展为齐次坐标：

\[
\mathbf p_h=
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
\]

齐次坐标最直接的优势是可以把平移也写成矩阵乘法：

\[
\begin{bmatrix}
1&0&0&t_x\\
0&1&0&t_y\\
0&0&1&t_z\\
0&0&0&1
\end{bmatrix}
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
=
\begin{bmatrix}
x+t_x\\
y+t_y\\
z+t_z\\
1
\end{bmatrix}
\]

点通常使用 \(w=1\)，方向向量通常使用 \(w=0\)。

当 \(w=0\) 时，平移量不会影响方向向量：

\[
\begin{bmatrix}
R&t\\
0&1
\end{bmatrix}
\begin{bmatrix}
\mathbf v\\0
\end{bmatrix}
=
\begin{bmatrix}
R\mathbf v\\0
\end{bmatrix}
\]

因此：

- 点具有位置，可以被平移；
- 向量只有方向和大小，不应该被平移。

齐次坐标还有一个更重要的作用：它让透视除法可以被纳入图形流水线。

---

## 三、模型矩阵 Model Matrix

模型矩阵描述物体在世界中的姿态。

通常可以由缩放、旋转和平移组合得到：

\[
M=TRS
\]

对于列向量，这意味着顶点实际依次经历：

```text
Scale → Rotate → Translate
```

因为最右边的矩阵最先作用。

矩阵乘法一般不可交换：

\[
TR\neq RT
\]

所以“先旋转再平移”和“先平移再旋转”通常会得到完全不同的结果。

---

### 绕 Z 轴旋转

绕 Z 轴旋转时，Z 坐标保持不变，只旋转 XY 平面。

旋转矩阵为：

\[
R_z(\theta)=
\begin{bmatrix}
\cos\theta&-\sin\theta&0&0\\
\sin\theta& \cos\theta&0&0\\
0&0&1&0\\
0&0&0&1
\end{bmatrix}
\]

C++ 三角函数使用弧度，因此需要将角度转换为弧度：

\[
\theta_{rad}
=
\theta_{degree}\frac{\pi}{180}
\]

Eigen 实现如下：

```cpp
Eigen::Matrix4f get_model_matrix(float rotation_angle)
{
    const float rad =
        rotation_angle * static_cast<float>(MY_PI) / 180.0f;

    Eigen::Matrix4f model;

    model << std::cos(rad), -std::sin(rad), 0, 0,
             std::sin(rad),  std::cos(rad), 0, 0,
             0,              0,             1, 0,
             0,              0,             0, 1;

    return model;
}
```

例如，将点 \((1,0,0)\) 绕 Z 轴旋转 \(90^\circ\)：

\[
x'=\cos90^\circ=0
\]

\[
y'=\sin90^\circ=1
\]

所以：

\[
(1,0,0)\rightarrow(0,1,0)
\]

---

## 四、观察矩阵 View Matrix

观察矩阵的作用不是“直接移动相机”，而是对整个世界应用相机变换的逆变换。

假设相机的世界变换为：

\[
C=T_cR_c
\]

那么观察矩阵就是：

\[
V=C^{-1}
\]

也就是：

\[
V=R_c^{-1}T_c^{-1}
\]

对于正交旋转矩阵：

\[
R^{-1}=R^T
\]

因此实际构造观察矩阵时，通常会使用相机旋转的转置以及相机位置的反向平移。

---

### 只有平移的简单情况

如果相机没有旋转，只位于：

\[
\mathbf e=(e_x,e_y,e_z)
\]

那么只需把整个世界平移：

\[
-\mathbf e
\]

观察矩阵为：

\[
V=
\begin{bmatrix}
1&0&0&-e_x\\
0&1&0&-e_y\\
0&0&1&-e_z\\
0&0&0&1
\end{bmatrix}
\]

代码如下：

```cpp
Eigen::Matrix4f get_view_matrix(Eigen::Vector3f eye_pos)
{
    Eigen::Matrix4f view;

    view << 1, 0, 0, -eye_pos.x(),
            0, 1, 0, -eye_pos.y(),
            0, 0, 1, -eye_pos.z(),
            0, 0, 0, 1;

    return view;
}
```

如果相机位于：

```text
eye = (0, 0, 5)
```

而某个顶点原来位于：

```text
(2, 0, -2)
```

观察变换之后：

```text
(2, 0, -2) - (0, 0, 5)
= (2, 0, -7)
```

这表示该顶点位于相机前方 7 个单位。

移动相机与反向移动整个世界，在最终图像上是等价的。

---

## 五、投影变换解决什么问题

观察空间仍然是三维空间，而屏幕是二维平面。

透视投影需要完成：

1. 产生近大远小的视觉效果；
2. 根据 FOV 和宽高比确定可见范围；
3. 将近远裁剪面映射到标准深度范围；
4. 为视锥裁剪和深度测试准备坐标。

透视投影的核心来自小孔相机模型和相似三角形。

---

## 六、使用相似三角形推导透视关系

假设相机位于原点并朝负 Z 轴观察。

观察空间中有一点：

\[
P=(x,y,z)
\]

相机前方的点满足：

\[
z<0
\]

根据相似三角形，投影后的坐标满足：

\[
x'\propto\frac{x}{-z}
\]

\[
y'\propto\frac{y}{-z}
\]

这已经体现出透视效果：

- 距离越近，\(-z\) 越小，投影越大；
- 距离越远，\(-z\) 越大，投影越小。

问题在于，普通矩阵乘法不能直接表达除法。

解决办法是先让：

\[
w_{clip}=-z
\]

然后由图形流水线统一进行透视除法。

---

## 七、通过齐次坐标实现除以 Z

为了得到：

\[
w_{clip}=-z
\]

可以把投影矩阵的最后一行设置为：

\[
\begin{bmatrix}
0&0&-1&0
\end{bmatrix}
\]

于是：

\[
\begin{bmatrix}
0&0&-1&0
\end{bmatrix}
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
=-z
\]

代码对应：

```cpp
projection(3, 2) = -1.0f;
```

经过投影矩阵后：

\[
w_{clip}=-z_{view}
\]

随后执行透视除法：

```cpp
vertex /= vertex.w();
```

就可以得到包含 \(1/(-z)\) 的坐标。

透视矩阵并没有在矩阵乘法阶段直接完成除法，而是把需要作为除数的 \(-z\) 暂存在 \(w\) 中，再交给后续的透视除法。

---

## 八、由 FOV 推导 Y 方向缩放

设垂直视野角为：

\[
\theta=\mathrm{FOV}_y
\]

在距离相机 \(d\) 的位置，视锥体顶部的高度为：

\[
t=d\tan\frac{\theta}{2}
\]

我们希望这个上边界在透视除法后映射到：

\[
y_{ndc}=1
\]

设 Y 方向缩放系数为 \(s\)，则：

\[
y_{clip}=sy
\]

又因为：

\[
w_{clip}=d=-z
\]

所以：

\[
y_{ndc}
=
\frac{sy}{d}
\]

将视锥顶部：

\[
y=d\tan\frac{\theta}{2}
\]

代入：

\[
1
=
\frac{
s d\tan(\theta/2)
}{
d
}
\]

消去 \(d\)：

\[
1=s\tan\frac{\theta}{2}
\]

因此：

\[
s
=
\frac{1}{
\tan(\theta/2)
}
\]

代码如下：

```cpp
const float fov_rad =
    eye_fov * static_cast<float>(MY_PI) / 180.0f;

const float scale =
    1.0f / std::tan(fov_rad / 2.0f);

projection(1, 1) = scale;
```

FOV 越大：

\[
\tan(\mathrm{FOV}/2)
\]

越大，因此 `scale` 越小，画面中的物体也越小，可见范围越广。

这与广角镜头的效果一致。

---

## 九、由宽高比推导 X 方向缩放

屏幕宽高比定义为：

\[
a=\frac{width}{height}
\]

在保持垂直 FOV 不变时，宽屏能够在水平方向显示更大的范围，因此 X 方向需要额外除以宽高比：

\[
x_{scale}=\frac{s}{a}
\]

所以：

```cpp
projection(0, 0) = scale / aspect_ratio;
projection(1, 1) = scale;
```

透视除法后的 XY 坐标为：

\[
x_{ndc}
=
\frac{s}{a}
\frac{x}{-z}
\]

\[
y_{ndc}
=
s
\frac{y}{-z}
\]

---

## 十、推导深度映射

XY 坐标决定点在屏幕上的位置，Z 坐标则用于：

- 近远平面裁剪；
- 深度测试；
- 遮挡关系判断。

设：

\[
n=zNear
\]

\[
F=zFar
\]

`zNear` 和 `zFar` 通常作为正距离传入。

因为相机朝负 Z 轴观察，所以它们在观察空间中的实际位置是：

\[
z_{near}=-n
\]

\[
z_{far}=-F
\]

在 OpenGL 风格的 NDC 中，希望：

\[
z=-n\rightarrow z_{ndc}=-1
\]

\[
z=-F\rightarrow z_{ndc}=1
\]

---

### 1. 假设裁剪空间 Z 的形式

设：

\[
z_{clip}=Az+B
\]

因为已经确定：

\[
w_{clip}=-z
\]

透视除法后：

\[
z_{ndc}
=
\frac{Az+B}{-z}
\]

现在只需要求出 \(A\) 和 \(B\)。

---

### 2. 代入近裁剪面

在近裁剪面：

\[
z=-n
\]

并且要求：

\[
z_{ndc}=-1
\]

代入：

\[
\frac{-An+B}{n}=-1
\]

整理得到：

\[
-An+B=-n
\]

---

### 3. 代入远裁剪面

在远裁剪面：

\[
z=-F
\]

并且要求：

\[
z_{ndc}=1
\]

代入：

\[
\frac{-AF+B}{F}=1
\]

整理得到：

\[
-AF+B=F
\]

---

### 4. 解出 A 和 B

联立：

\[
-An+B=-n
\]

\[
-AF+B=F
\]

两式相减：

\[
-A(F-n)=F+n
\]

因此：

\[
A=-\frac{F+n}{F-n}
\]

再代回原式：

\[
B=-\frac{2Fn}{F-n}
\]

所以深度部分为：

\[
z_{clip}
=
-\frac{F+n}{F-n}z
-
\frac{2Fn}{F-n}
\]

代码对应：

```cpp
projection(2, 2) =
    -(zFar + zNear) / (zFar - zNear);

projection(2, 3) =
    -(2.0f * zFar * zNear) / (zFar - zNear);
```

---

## 十一、完整透视投影矩阵

令：

\[
s=\frac{1}{\tan(\mathrm{FOV}_y/2)}
\]

\[
a=\mathrm{aspect}
\]

最终得到：

\[
P=
\begin{bmatrix}
\frac{s}{a}&0&0&0\\
0&s&0&0\\
0&0&-\frac{F+n}{F-n}&-\frac{2Fn}{F-n}\\
0&0&-1&0
\end{bmatrix}
\]

Eigen 实现：

```cpp
Eigen::Matrix4f get_projection_matrix(
    float eye_fov,
    float aspect_ratio,
    float zNear,
    float zFar)
{
    const float fov_rad =
        eye_fov * static_cast<float>(MY_PI) / 180.0f;

    const float scale =
        1.0f / std::tan(fov_rad / 2.0f);

    Eigen::Matrix4f projection =
        Eigen::Matrix4f::Zero();

    projection(0, 0) = scale / aspect_ratio;
    projection(1, 1) = scale;

    projection(2, 2) =
        -(zFar + zNear) / (zFar - zNear);

    projection(2, 3) =
        -(2.0f * zFar * zNear) /
        (zFar - zNear);

    projection(3, 2) = -1.0f;

    return projection;
}
```

其中：

```cpp
projection(row, column)
```

表示访问 Eigen 矩阵的指定行和指定列，索引从 0 开始。

也可以使用 Eigen 的逗号初始化器一次写完整个矩阵：

```cpp
projection <<
    scale / aspect_ratio, 0, 0, 0,
    0, scale, 0, 0,
    0, 0,
    -(zFar + zNear) / (zFar - zNear),
    -(2.0f * zFar * zNear) / (zFar - zNear),
    0, 0, -1.0f, 0;
```

---

## 十二、透视矩阵作用后的结果

将观察空间点：

\[
\mathbf p_{view}
=
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
\]

乘以投影矩阵：

\[
\mathbf p_{clip}=P\mathbf p_{view}
\]

得到：

\[
x_{clip}
=
\frac{s}{a}x
\]

\[
y_{clip}=sy
\]

\[
z_{clip}
=
-\frac{F+n}{F-n}z
-
\frac{2Fn}{F-n}
\]

\[
w_{clip}=-z
\]

透视除法之后：

\[
x_{ndc}
=
\frac{s}{a}
\frac{x}{-z}
\]

\[
y_{ndc}
=
s
\frac{y}{-z}
\]

\[
z_{ndc}
=
\frac{
-\frac{F+n}{F-n}z-\frac{2Fn}{F-n}
}{
-z
}
\]

XY 中出现了 \(1/(-z)\)，所以产生近大远小。

---

## 十三、为什么深度不是线性的

整理深度公式：

\[
z_{ndc}
=
\frac{F+n}{F-n}
+
\frac{2Fn}{(F-n)z}
\]

其中存在：

\[
\frac{1}{z}
\]

所以深度值相对于观察空间距离不是线性的。

结果是：

- 靠近近裁剪面的区域拥有更高深度精度；
- 越接近远裁剪面，深度值越集中；
- `zNear` 设置得过小会浪费大量深度精度；
- 深度精度不足时可能产生 Z-fighting。

因此，不应该在没有必要时把近裁剪面设置为极小值。

例如：

```text
near = 0.1
far  = 1000
```

通常比：

```text
near = 0.001
far  = 1000
```

拥有更好的有效深度精度。

---

## 十四、透视除法

得到裁剪坐标后，光栅化器执行：

```cpp
for (auto& vertex : vertices)
{
    vertex /= vertex.w();
}
```

即：

\[
(x,y,z,w)
\rightarrow
\left(
\frac{x}{w},
\frac{y}{w},
\frac{z}{w},
1
\right)
\]

透视除法完成后得到 NDC。

理论上，裁剪应在透视除法之前、裁剪空间中完成，因为三角形可能部分穿过近裁剪面。如果过早除以 \(w\)，当顶点接近相机平面、\(w\approx0\) 时，坐标会变得非常大甚至出现数值异常。

---

## 十五、视口变换

NDC 的 XY 范围是：

\[
[-1,1]
\]

需要将其映射到实际屏幕尺寸。

对于宽度 \(W\)、高度 \(H\)：

\[
x_{screen}
=
\frac{W}{2}(x_{ndc}+1)
\]

\[
y_{screen}
=
\frac{H}{2}(y_{ndc}+1)
\]

代码形式：

```cpp
vertex.x() =
    0.5f * width * (vertex.x() + 1.0f);

vertex.y() =
    0.5f * height * (vertex.y() + 1.0f);
```

对应关系：

```text
NDC x=-1 → 屏幕左侧
NDC x= 0 → 屏幕中心
NDC x= 1 → 屏幕右侧
```

图形学数学坐标通常以左下角为原点、Y 轴向上，而图像数组通常以左上角为原点、Y 轴向下，因此写入帧缓冲时还可能需要翻转 Y 轴：

```cpp
buffer_index =
    (height - 1 - y) * width + x;
```

---

## 十六、用一个实际顶点验证整个过程

假设参数为：

```text
FOV    = 45°
aspect = 1
near   = 0.1
far    = 50
```

那么：

\[
s=
\frac{1}{\tan22.5^\circ}
\approx2.414
\]

某个模型顶点为：

\[
(2,0,-2)
\]

相机位于：

\[
(0,0,5)
\]

并且模型没有旋转。

---

### 1. 模型变换

模型矩阵为单位矩阵：

\[
\mathbf p_{world}=(2,0,-2)
\]

### 2. 观察变换

世界整体平移 \((0,0,-5)\)：

\[
\mathbf p_{view}=(2,0,-7)
\]

### 3. 投影与透视除法

X 方向的 NDC 坐标为：

\[
x_{ndc}
=
2.414\frac{2}{7}
\approx0.69
\]

Y 坐标为：

\[
y_{ndc}=0
\]

### 4. 视口变换

屏幕大小为 \(700\times700\)：

\[
x_{screen}
=
350(0.69+1)
\approx592
\]

\[
y_{screen}
=
350(0+1)
=
350
\]

因此该顶点会出现在屏幕中心右侧，大约为：

```text
(592, 350)
```

这个过程展示了一个三维顶点如何经过 MVP 和视口变换，最终落到具体像素位置。

---

## 十七、MVP 的矩阵顺序

代码通常写成：

```cpp
Eigen::Matrix4f mvp =
    projection * view * model;

Eigen::Vector4f clip_position =
    mvp * local_position;
```

对于列向量，实际执行顺序为：

```text
local_position
    ↓ model
world_position
    ↓ view
view_position
    ↓ projection
clip_position
```

也就是：

\[
\mathbf p_{clip}
=
P(V(M\mathbf p_{local}))
\]

不能写成：

```cpp
model * view * projection
```

因为矩阵乘法一般不满足交换律。

---

## 十八、MVP 之后还有哪些步骤

MVP 只负责把顶点转换到裁剪空间，完整图形流水线还包括：

1. 裁剪；
2. 透视除法；
3. 视口变换；
4. 图元装配；
5. 三角形光栅化；
6. 重心坐标计算；
7. 属性插值；
8. 深度测试；
9. 片元着色；
10. 写入帧缓冲。

在简单的线框作业中，程序只需要将三个顶点转换到屏幕空间，然后使用直线算法连接：

```text
v0 → v1
v1 → v2
v2 → v0
```

后续实现实心三角形时，还需要判断像素是否位于三角形内部，并使用重心坐标插值深度、颜色、法线和纹理坐标。

---

## 十九、常见错误

### 1. 忘记把角度转换成弧度

错误：

```cpp
std::cos(rotation_angle);
```

正确：

```cpp
const float rad = rotation_angle * PI / 180.0f;
std::cos(rad);
```

### 2. MVP 顺序写反

对于列向量，应使用：

```cpp
projection * view * model * vertex
```

### 3. 使用单位矩阵初始化投影矩阵

透视矩阵中大量元素应该为 0，因此更安全的写法是：

```cpp
Eigen::Matrix4f projection =
    Eigen::Matrix4f::Zero();
```

然后只填写非零元素。

### 4. 忘记透视除法

只乘投影矩阵并不会直接得到 NDC，还必须执行：

```cpp
vertex /= vertex.w();
```

### 5. 混淆 near/far 的距离和 Z 坐标

通常传入：

```text
near > 0
far  > 0
```

但在相机朝负 Z 轴的观察空间中：

```text
近裁剪面位于 z=-near
远裁剪面位于 z=-far
```

### 6. 混用不同图形 API 的投影矩阵

OpenGL 传统 NDC 深度范围为：

\[
[-1,1]
\]

Direct3D 和常见 Vulkan 配置通常使用：

\[
[0,1]
\]

另外，不同 API 对 Y 轴方向也可能有不同约定。矩阵公式不能脱离坐标系和 API 约定单独讨论。

### 7. 近裁剪面设置得太小

过小的 `zNear` 会严重降低有效深度精度，增加 Z-fighting 风险。

### 8. 屏幕 Y 轴翻转出现越界

如果屏幕高度为 `height`，合法行号是：

```text
0 到 height-1
```

因此翻转 Y 轴通常应使用：

```cpp
height - 1 - y
```

而不是：

```cpp
height - y
```

---

## 二十、总结

MVP 变换可以概括为：

\[
\boxed{
\mathbf p_{clip}
=
\mathbf P\mathbf V\mathbf M\mathbf p_{local}
}
\]

其中：

- \(M\) 把物体从局部空间放到世界空间；
- \(V\) 把世界变换到相机空间；
- \(P\) 把视锥体变换到裁剪空间。

透视投影最核心的设计是：

\[
\boxed{w_{clip}=-z_{view}}
\]

后续通过透视除法：

\[
\boxed{
x_{ndc}=\frac{x_{clip}}{w_{clip}},
\quad
y_{ndc}=\frac{y_{clip}}{w_{clip}}
}
\]

得到：

\[
x_{ndc}\propto\frac{x}{-z}
\]

\[
y_{ndc}\propto\frac{y}{-z}
\]

于是自然产生近大远小。

完整透视矩阵为：

\[
\boxed{
P=
\begin{bmatrix}
\frac{1}{a\tan(\theta/2)}&0&0&0\\
0&\frac{1}{\tan(\theta/2)}&0&0\\
0&0&-\frac{F+n}{F-n}&-\frac{2Fn}{F-n}\\
0&0&-1&0
\end{bmatrix}
}
\]

它不是一个需要死记硬背的公式，而是由以下条件共同决定的：

1. 相似三角形要求屏幕位置与 \(x/(-z)\)、\(y/(-z)\) 成正比；
2. FOV 决定垂直可见范围；
3. Aspect Ratio 决定水平可见范围；
4. 齐次坐标让 \(w=-z\)，从而可以进行透视除法；
5. 近远裁剪面决定深度映射的两个边界条件。

理解这五点，就可以从头推导出整个透视投影矩阵，而不必机械记忆矩阵中的每一个元素。
