---
title: Mircofacet and Kulla-Conty Approximation
categories: 渲染
tags:
  - RTR
  - PBR
mathjax: true
---

最近刚做完GAMES202的homework4，对于PBR、Microfacet模型存在一些疑问，故整理成博客以供阅读。

## PBR

PBR(Physically Based Rendering)即一组基于物理原理、尽可能匹配真实物理世界的一组渲染方法，相较于普通方法更加符合物理原理且更加真实。离线渲染的PBR详见[pbrt](https://www.pbr-book.org/)，这里仅讨论实时渲染PBR。

![Disney's Hyperion Renderer](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210707172110505.png)




在实时渲染领域，PBR表面材质主要包括Microfacet model（微表面模型）以及Disney Principled BRDFs（Most for artists），然而实际上两者并非严格意义上的**基于物理的渲染**，大多还是对物理规则的近似。

本篇博客主要考虑实时渲染下的Mircofacet模型如何实现PBR，需要满足如下要求：

1. 基于Microfacet模型
2. 保持能量守恒​​
3. Physically-based BRDF
4. 实时级别的性能

<!--more-->

## Microfacet Model

通常情况下我们将物体表面的反射情况分为diffuse、specular以及两者之间的glossy情况：



![Diffuse, Mirror, Glossy](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210708094742444.png)



然而Microfacet Model通常被认为是对物理世界物体表面的完美模拟，Microfacet Model认为物体表面是由很多Microfacet（微表面）集合构成的，在这些Microfacet上服从**镜面反射**原理。根据微元法的思想，在足够小的Microfacet上使用镜面反射原理去计算，那么累加起来后的结果也会足够接近真实结果，因此我们可以认为Microfacet Model是一种相对准确的模型。



![Microfacet surface](https://learnopengl.com/img/pbr/microfacets_light_rays.png)



将表面划分的足够细后，如何区分不同粗糙程度的表面呢？显而易见的是，根据表面的粗糙系数可构建Microfacet模型，即越粗糙理论上微表面应该越多且越“坑坑洼洼”，那么进一步“坑坑洼洼”如何体现呢？如果把每个微表面看作镜面，那么法向量可以确定这个镜面，粗略估计下来，越粗糙的平面意味着这些法向量的分布越不集中，反之则越集中。这些法向量同样也是每个微表面上的入射光线和出射光线的`halfway vector`，定义为$h$，$h$的具体计算方式如下：
$$
\begin{equation}
h = \frac{l+v}{||l+v||} 
\end{equation}
$$

下图是`roughness`从0.1递增至1.0的中间结果：

![Visualized NDF (Normalized Distribution Function)](https://learnopengl.com/img/pbr/ndf.png)



### Microfaect BRDF

实际上，对于Microfacet模型，我们几乎不会将之视为很多表面来具体计算，一方面这些Microfacet过小以至于我们很难观察到它的细节，另一方面过多的Microfacet会大大增加计算量，实际使用中常常使用Macrosurface来代替Microsurface，见下图：


![Micro v.s. marco surface](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210708104047492.png)

在Macrosurface中，我们会使用microfacet分布函数$D$，shadowing-masking函数$G$，再加上Fresnel term$F$共同构成Microfacet BRDF，如下：
$$
f(i,o)=\frac{F(i,h)G(i,o,h)D(h)}{4(n,i)(n,o)}
$$


下面分别介绍$F$，$G$，$D$

### Fresnel term

菲涅尔项（Fresnel term）描述了不同观察角度的表面反射情况，以下图为例，**观察角度（和平面法线夹角）越大**，木板地面的反射就越清晰（和光的偏振有关）。

![Reflectance increase with gazing angle](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210708112109987.png)

菲涅尔项的计算公式是十分复杂的：
$$
R_{\mathrm{s}}=\left|\frac{n_{1} \cos \theta_{\mathrm{i}}-n_{2} \cos \theta_{\mathrm{t}}}{n_{1} \cos \theta_{\mathrm{i}}+n_{2} \cos \theta_{\mathrm{t}}}\right|^{2}=\left|\frac{n_{1} \cos \theta_{\mathrm{i}}-n_{2} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}}}{n_{1} \cos \theta_{\mathrm{i}}+n_{2} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}}}\right|^{2} 
$$

$$
R_{\mathrm{p}}=\left|\frac{n_{1} \cos \theta_{\mathrm{t}}-n_{2} \cos \theta_{\mathrm{i}}}{n_{1} \cos \theta_{\mathrm{t}}+n_{2} \cos \theta_{\mathrm{i}}}\right|^{2}=\left|\frac{n_{1} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}-n_{2} \cos \theta_{\mathrm{i}}}}{n_{1} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}}+n_{2} \cos \theta_{\mathrm{i}}}\right| 
$$

反射比是两者的算术平均值：

$$
R_{\mathrm{eff}}=\frac{1}{2}\left(R_{\mathrm{s}}+R_{\mathrm{p}}\right) \label{5}
$$


通常情况下我们采用`Schlick's approximation`对菲涅尔项进行估计，即光从折射率为$n_1$的介质向另一种折射率为$n_2$的介质传播时，菲涅尔项有：


$$
\begin{equation}
	R(\theta)=R_{0}+\left(1-R_{0}\right)(1-\cos \theta)^{5} 
\end{equation}
$$


$$
\begin{equation}
	R_0=\left(\frac{n_1-n_2}{n_1+n_2}\right)^{2} 
\end{equation}
$$

如此代码的实现就非常简单了，这里以`glsl`为例：

```glsl
vec3 fresnelSchlick(vec3 R0, vec3 V, vec3 H)
{
    return R0 + (1.0 - R0) * pow((1.0 - max(0.0001, dot(V, H))), 5.0);
}
```

对于`Schlick's approximation`，$R_0$项通常需要我们自己取，对绝缘体来说$R_0$可以取0.04，对导体而言$R_0$可以取0.92。

总结，菲涅尔项的`Schlick's approximation`取决于**光线发射的介质与穿过的介质**以及**观察向量与平面法向量**的夹角。









## 参考资料

[^1]: https://www.graphics.cornell.edu/~bjw/microfacetbsdf.pdf
[^2]: https://learnopengl.com/PBR/Theory
[^3]: https://sites.cs.ucsb.edu/~lingqi/teaching/games202.html

