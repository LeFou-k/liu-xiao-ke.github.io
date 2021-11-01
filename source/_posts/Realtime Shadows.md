---
title: Realtime Shadows
categories: 渲染
tags:
  - RTR
date: 2021/10/29
updated: 2021/10/29
mathjax: true
---

实时阴影是真实感渲染中非常重要的一部分，在离线渲染中，我们对每个shading point进行MIS(Multiple Importance Sampling)，如果是Area Light光源，基于采样自然会形成十分不错的软阴影，如下图所示：

![cbox_mis rendered by Nori](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/cbox_mis.png)



然而实际上在实时渲染的过程中，采样每道光线是一件十分奢侈的事情，即无法通过重要性采样的方法来求解渲染方程，这是我们在理解所有实时渲染算法与离线渲染算法的关键区别。

<!--more-->

首先来理解一下ShadowMap背后的数学原理，即为何要引入ShadowMap以及引入ShadowMap后如何帮助我们求解渲染方程。

## Math Behind

实时渲染中渲染方程的求解通常会利用如下不等式来近似：
$$
\begin{equation}
\int_{\Omega} f(x) g(x) \mathrm{d} x \approx \frac{\int_{\Omega} f(x) \mathrm{d} x}{\int_{\Omega} \mathrm{d} x} \cdot \int_{\Omega} g(x) \mathrm{d} x
\label{1}
\end{equation}
$$

需要注意该不等式的适用条件如下：

- $\mathrm{d}x$积分区间足够小
- 或$g(x)$函数在积分区间上足够平滑

后续在`Split sum`以及`Ambient Occulusion`中会继续用到该不等式。

另外，渲染方程如下：
$$
\begin{equation}
L_{o}\left(\mathrm{p}, \omega_{o}\right)=\int_{\Omega^{+}} L_{i}\left(\mathrm{p}, \omega_{i}\right) f_{r}\left(\mathrm{p}, \omega_{i}, \omega_{o}\right) \cos \theta_{i} V\left(\mathrm{p}, \omega_{i}\right) \mathrm{d} \omega_{i}
\label{2}
\end{equation}
$$
基于$\eqref{1}$式以及$\eqref{2}$式，我们将`Visibility`项提出后可得到如下转化：

$$
\begin{equation}
L_{o}\left(\mathrm{p}, \omega_{o}\right) \approx \frac{\int_{\Omega^{+}} V\left(\mathrm{p}, \omega_{i}\right) \mathrm{d} \omega_{i}}{\int_{\Omega^{+}} \mathrm{d} \omega_{i}} \cdot \int_{\Omega^{+}} L_{i}\left(\mathrm{p}, \omega_{i}\right) f_{r}\left(\mathrm{p}, \omega_{i}, \omega_{o}\right) \cos \theta_{i} \mathrm{~d} \omega_{i}
\label{3}
\end{equation}
$$

对于Constant Randiance的Area light以及diffuse BRDF来说，$\eqref{3}$式中$g(x)$项足够平滑，因此满足不等式的适用条件。

此时来看拆分出来的$\frac{\int_{\Omega^{+}} V\left(\mathrm{p}, \omega_{i}\right) \mathrm{d} \omega_{i}}{\int_{\Omega^{+}} \mathrm{d} \omega_{i}}$项，如果将积分视为求和的话，该项相当于求解$\mathrm{d}\omega_i$范围内的$V$项平均值，即我们需要求解shading point周围点的平均可见度(Visibility)，由此我们引申出Shadow map的概念。 



## Shadow map

一言概之，Shadow map即把light视为相机后求解得到的深度缓存，Shadow map每个像素存放相应点的深度。可以理解为离线渲染中shadowRay的逆向过程。

求解shadow map的伪代码如下：

```c++
Vec3 lightpos; 
Matrix4x4 MVP; //view lightpos as camera pos
for every objects:
	newposition = MVP * oldposition
for pixel p in shadowmap:
	trace a ray from lightpos to p;
	get the tmin and save in shadowmp;
```

![shadowmap(图源:Realtime-Rendering)](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/shadowMap.png)

### Artifacts

ShadowMap求解shading point阴影的核心是比较shadowRay与shadowMap交点处的值与当前shading point的深度。因此如果比较部分不够准确则会出现Artifacts

基于ShadowMap渲染出的阴影效果主要由以下两点决定：

1. ShadowMap的分辨率

   由于ShadowMap是存在分辨率的，每个ShadowMap pixel存放当前pixel还原到世界坐标中的surface对应的depth，因此当ShadowMap的分辨率不足时会导致pixel还原到世界中的surface覆盖范围过大，此时ShadowMap中的深度值不准确，也就是说本应该比该pixel“浅”的shading point可能会被遮挡，即"Self occlusion"现象，见下左图。

2. z-buffer的数值精度

   z-buffer的原理就很简单了，在比较深度时由于浮点数的精度问题导致比较产生相反的结果也会出现"Self Occlusion"类似的Artifacts

![ShadowMap's Artifacts](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/shadowMap's artifacts.png)

解决方法即加上一个`bias factor`，这也是渲染领域中解决浮点数精度问题的常用方法，然而如上右图所示，如果`bias factor`调整加入后不改变较低的ShadowMap resolution，会出现**上右图**的Artifacts，即"detached shadows"

另外，常用的解决Artifacts的方法见下图，左边即常用的存储第一接触面的深度，中间是另一种常见的避免"self-occlusion"，即存储第二接触面的深度，右边为两者的折中办法，存放中间接触面的深度等等，具体细节这里不过多赘述。

![Artifacts' solution](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/ShadowMap's artifacts solution.png)





## Hard shadow

如上图，有了Shadow Map，我们可以对点光源很轻松的渲染出影阴影，核心思想即检查shading point处的Visibility即可，伪代码如下所示：

```c++
Vec3 lightpos;
texture2d shadowMap;
for every shading points:
	trace a ray from shading point to lightpos;
	get the intersection pixel of ray and shadowMap;
	if(shadowMap(intersection pixel) < distance from lightpos to shadingpos)
        render black;
	else render normal color;
```

然而实际上对于Arealight，此时的计算方法不再适用，以下图为示：

![umbra and penumbra](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/umbra.png)

因为对于点光源而言，每个shading point只有`visible`和`invisible`两种状态，阴影的边界是十分明显的，然而对于Area light而言，只有图中蓝色部分可称为被occlusion完全遮挡，其他红色部分只是半遮挡，我们分别命名为`umbra`（全影）和`penumbra`（半影）。

Soft shadow相关方法能够更加真实的渲染阴影。



## Soft shadow

进一步，基于我们上述的数学推导以及渲染原理的概述，此时对软阴影的渲染应该有了大概的把握，核心原理即找到shading point点处的`visibility`，由上图亦可知，shading point的`visibility`应该等价于check其周围点`visibility`的均值，即滤波的方法。本质上这些方法的核心都是离线渲染中ShadowRay的逆向trace过程，通过ShadowRay trace到的点进行采样从而估计当前Shading Point的visibility

后续PCF、PCSS、VSSM、MSM等方法，一言蔽之，PCF即固定滤波核的大小；PCSS用一个和shading point到blocker距离成反比的可变滤波核大小来滤波；VSSM则用来简化这个过程，通过将shading point周围点的depth大小用正态分布来描述从而避免滤波；MSM则进一步将VSSM中的正态分布推广，使其更加接近实际的PCF中的Depth分布。

### PCF

前面铺垫了这么多，PCF整体的思想就很好理解了。PCF(Percentage Closer Filtering)顾名思义，即采样周围的点并计算shadingPoint相对这些样本visible的比例。需要注意的是，PCF是对采样点visibility的滤波，而非对Shadow Map的滤波，有如下几个原因：

1. 对Shadow Map滤波并没有实际意义，滤波后仅会得到一张blurred的Shadow Map，再进行比较得到的结果仍然是0/1的visibility，所以求解得到的阴影仍然是边缘锐利的，无益于Soft shadow
2. 如$\eqref{3}$式所示，基于数学原理来看也是对$V$项的滤波。

PCF的伪代码如下所示：

```c++
for every pixel in samples:
	if(shadingPoint.depth < pixel.depth) //this pixel is visible
        add this pixel to final percentage
```

PCF的伪代码是非常简单的，事实上PCF并非Physical-Based，因为没有利用到Shading point到周围blocker的距离，意味着无论Shading Point与blocker距离多远，penumbra(半影)的大小是相同的，如下图所示。

<img src="https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/PCF.png" alt="image-20211031234155865" style="zoom:67%;" />

然而实际上，Physical-Based的阴影是距离Blocker越近越Shaper，相反则越Softer，直觉上阴影应该和Shading point到Blocker的距离有关，由此引出PCSS的算法。

### PCSS



### VSSM

### MSM

