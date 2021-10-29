---
title: Realtime Shadows
categories: 渲染
tags:
  - RTR
date: 2021/10/29
updated: 2021/10/29
mathjax: trues
---

实时阴影是真实感渲染中非常重要的一部分，在离线渲染中，我们对每个shading point进行MIS(Multiple Importance Sampling)，如果是Area Light光源，基于采样自然会形成十分不错的软阴影，如下图所示：

![cbox_mis rendered by Nori](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/cbox_mis.png)



然而实际上在实时渲染的过程中，采样每道光线是一件十分奢侈的事情，即无法通过重要性采样的方法来求解渲染方程，这是我们在理解所有实时渲染算法与离线渲染算法的关键区别。

<!--more-->

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

一言概之，Shadow map即把light视为相机后求解得到的Z-buffer，Shadow map每个像素存放相应点的深度。求解shadow map的伪代码如下：

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

因为对于点光源而言，每个shading point只有`visible`和`invisible`两种状态，阴影的边界是十分明显的，然而对于Area light而言，只有图中蓝色部分可称为被occlusion完全遮挡，其他红色部分只是半遮挡，我们分别命名为`umbra`（全影）和`penumbra`（半影），顾名思义。

Soft shadow相关方法能够更加真实的渲染实际的阴影。



## Soft shadow

进一步，基于我们上述的数学推导

### PCF

### PCSS

### VSSM

### MSM

