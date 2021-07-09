---
title: Mircofacet and Kulla-Conty Approximation
categories: 渲染
tags:
  - RTR
  - PBR
date: 2021/7/8
updated: 2021/7/9
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
h = \frac{l+v}{||l+v||} \label{1}
\end{equation}
$$

下图是`roughness`从0.1递增至1.0的中间结果：

![Visualized NDF (Normalized Distribution Function)](https://learnopengl.com/img/pbr/ndf.png)



### Microfaect BRDF

实际上，对于Microfacet模型，我们几乎不会将之视为很多表面来具体计算，一方面这些Microfacet过小以至于我们很难观察到它的细节，另一方面过多的Microfacet会大大增加计算量，实际使用中常常使用Macrosurface来代替Microsurface，见下图：


![Micro v.s. marco surface](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210708104047492.png)

在Macrosurface中，我们会使用microfacet分布函数$D$，shadowing-masking函数$G$，再加上Fresnel term$F$共同构成Microfacet BRDF，利用Macroface以及这三个参数足以描述Microface模型，如下：
$$
\begin{equation}
f(\boldsymbol{i},\boldsymbol{o})=\frac{F(\boldsymbol{i},\boldsymbol{h})G(\boldsymbol{i},\boldsymbol{o},\boldsymbol{h})D(\boldsymbol{h})}{4(\boldsymbol{n},\boldsymbol{i})(\boldsymbol{n},\boldsymbol{o})} \label{2}
\end{equation}
$$

下面分别介绍$F$，$G$，$D$



### Fresnel term

菲涅尔项（Fresnel term）描述了不同观察角度的表面反射情况，以下图为例，**观察角度（和平面法线夹角）越大**，木板地面的反射就越清晰（和光的偏振有关）。

![Reflectance increase with gazing angle](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/image-20210708112109987.png)

菲涅尔项的计算公式是十分复杂的：
$$
\begin{equation}
R_{\mathrm{s}}=\left|\frac{n_{1} \cos \theta_{\mathrm{i}}-n_{2} \cos \theta_{\mathrm{t}}}{n_{1} \cos \theta_{\mathrm{i}}+n_{2} \cos \theta_{\mathrm{t}}}\right|^{2}=\left|\frac{n_{1} \cos \theta_{\mathrm{i}}-n_{2} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}}}{n_{1} \cos \theta_{\mathrm{i}}+n_{2} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}}}\right|^{2} \label{3}
\end{equation}
$$

$$
\begin{equation}
R_{\mathrm{p}}=\left|\frac{n_{1} \cos \theta_{\mathrm{t}}-n_{2} \cos \theta_{\mathrm{i}}}{n_{1} \cos \theta_{\mathrm{t}}+n_{2} \cos \theta_{\mathrm{i}}}\right|^{2}=\left|\frac{n_{1} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}-n_{2} \cos \theta_{\mathrm{i}}}}{n_{1} \sqrt{1-\left(\frac{n_{1}}{n_{2}} \sin \theta_{\mathrm{i}}\right)^{2}}+n_{2} \cos \theta_{\mathrm{i}}}\right| \label{4}
\end{equation}
$$

反射比是两者的算术平均值：

$$
\begin{equation}
R_{\mathrm{eff}}=\frac{1}{2}\left(R_{\mathrm{s}}+R_{\mathrm{p}}\right) \label{5}
\end{equation}
$$


通常情况下我们采用`Schlick's approximation`对菲涅尔项进行估计，即光从折射率为$n_1$的介质向另一种折射率为$n_2$的介质传播时，菲涅尔项有：


$$
\begin{equation}
	R(\theta)=R_{0}+\left(1-R_{0}\right)(1-\cos \theta)^{5} \label{6}
\end{equation}
$$


$$
\begin{equation}
	R_0=\left(\frac{n_1-n_2}{n_1+n_2}\right)^{2} \label{7}
\end{equation}
$$

如此代码的实现就非常简单了，由公式$\eqref{6}$和$\eqref{7}$这里以`glsl`为例：

```glsl
vec3 fresnelSchlick(vec3 R0, vec3 V, vec3 H)
{
    return R0 + (1.0 - R0) * pow((1.0 - max(0.0001, dot(V, H))), 5.0);
}
```


对于`Schlick's approximation`，$R_0$项通常需要我们自己取，对绝缘体来说$R_0$可以取0.04，对导体而言$R_0$可以取0.92。

总结，菲涅尔项的`Schlick's approximation`取决于**光线发射的介质与穿过的介质**以及**观察向量与平面法向量**的夹角。



### Normal Distribution Function

法向量分布函数（Normal Distribution Function）简称NDF，注意与正态分布无关，但呈现形态类似正态分布，描述了微观几何平面上的法线分布的统计概率，是决定Microfacet Model呈现效果的关键。

NDF存在多种模型，大多与正态分布形态相近，呈现类似高斯分布的“中间高、两头低”的形态。从图形学角度来理解，无论微表面如何划分，其总体仍是围绕Macroface的法向量（后面皆称为$N$）分布的，且理论上和$N$偏离角度越大则分布越稀少。

NDF具有一些共同性质，首先给定积分域表面微元$dA$，需满足如下积分情况：
$$
\begin{equation}
\int_{\mathrm{H}^{2}(\mathbf{n})} D\left(\omega_{\mathrm{h}}\right) \mathrm{dA}=1\label{8}
\end{equation}
$$

接着由立体角和投影立体角之间的关系$\mathrm{dA}=\mathrm{d} \omega_\mathrm{h}\cos{\theta_\mathrm{h}}$进一步可将其转化为：

$$
\begin{equation}
\int_{\mathrm{H}^{2}(\mathbf{n})} D\left(\omega_{\mathrm{h}}\right) \cos \theta_{\mathrm{h}} \mathrm{d} \omega_{\mathrm{h}}=1 \label{9}
\end{equation}
$$

因此所有无论如何定义的$\mathrm{D(\omega_\mathrm{h})}$均应该满足$\eqref{9}$式的要求，即`Normalize`（归一化），后续主要通过`Beckmann NDF`以及`GGX NDF`两者之间的对比来看NDF的具体形式如何。

其次，NDF一般为`roughness`$\alpha$和半程向量与$N$之间的夹角$\theta$两者的函数，且函数大体与高斯分布形式类似。



#### Beckmann NDF

`Beckmann NDF`的计算公式如下，$\alpha$为物体表面粗糙系数，$\mathrm{\theta_{\mathrm{h}}}$为$h$与$N$之间的夹角。
$$
\begin{equation}
D(h)=\frac{e^{-\frac{\tan ^{2} \theta_{h}}{\alpha^{2}}}}{\pi \alpha^{2} \cos ^{4} \theta_{h}} \label{10}
\end{equation}
$$

#### GGX NDF

`GGX NDF`的计算公式如下：
$$
\begin{equation}
D_{G G X}(\boldsymbol{h})=\frac{\alpha^{2}}{\pi\left((\boldsymbol{n} \cdot \boldsymbol{h})^{2}\left(\left(\alpha^{2}-1\right)+1\right)\right)^{2}} \label{11}
\end{equation}
$$


其中$\alpha=roughness^2$

GGX相比于Backmann的一个显著特征是拥有更加长的“尾巴”，此外，可将GGX进一步推广为GTR，在此不过多赘述，详见[GAMES202 lecture10](https://sites.cs.ucsb.edu/~lingqi/teaching/resources/GAMES202_Lecture_10.pdf)。

以`GGX NDF`为例，实现代码如下所示：

```glsl
float DistributionGGX(vec3 N, vec3 H, float roughness)
{
    float a = roughness * roughness;
    float a2 = a * a;
    float NdotH = max(dot(N, H), 0.0);
    float NdotH2 = NdotH * NdotH;

    float nom = a2;
    float denom = (NdotH2 * (a2 - 1.0) + 1.0);
    denom = PI * denom * denom;
    return nom / max(denom, 0.0001);
}
```

需要注意的是，上述实现过程中多次使用max与0.0001比较，这是因为$D(h)>0$恒成立，即无论$h$与$N$的偏离角度多大，都理论上存在相应的微表面，符合实际，使用max避免返回负值导致结果失去物理意义。



#### Comparison

比对`Beckmann NDF`和`GGX NDF`如下图所示：

![Beckmann(red), Phong(blue), GGX(green) distribution functions](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/20210708180459.png)



对比来看，GGX最显著的特点即“longer tail”，表现在实际中即高光的过渡效果会更好，看起来更符合实际：

<img src="https://planetside.co.uk/wp-content/uploads/2020/11/specmodels_v2_Conductive_FrontLit_Socials-450x267.jpg" alt="Specular roughness models on conductor surface." style="zoom:150%;" />



### Shadow-masking function

可以清楚的发现，公式$\eqref{2}$中的BRDF分母项为$(\boldsymbol{n},\boldsymbol{i})(\boldsymbol{n},\boldsymbol{o})$，当camera与shading point向量与$N$呈90度夹角时，即$(\boldsymbol{n},\boldsymbol{i})\to0$，此时会导致BRDF无限大，会渲染出来一条“白边”，为避免这种现象从而引入了shadow-masking function $G$.

常有的shadow-masking function为Smith模型：
$$
\begin{equation}
G_{S m i t h}(\boldsymbol{i}, \boldsymbol{o}, \boldsymbol{h})=G_{\text {Schlick }}(\boldsymbol{l}, \boldsymbol{h}) G_{\text {Schlick }}(\boldsymbol{v}, \boldsymbol{h}) \label{12}
\end{equation}
$$


$$
\begin{equation}
G_{S c h l i c k}(\boldsymbol{v}, \boldsymbol{n})=\frac{\boldsymbol{n} \cdot \boldsymbol{v}}{\boldsymbol{n} \cdot \boldsymbol{v}(1-k)+k} \label{13}
\end{equation}
$$

其中$k=\frac{(roughness+1)^2}{8}$

实现方式如下：

```glsl
float GeometrySchlickGGX(float NdotV, float roughness) {
    float a = roughness;
    float k = (a + 1)*(a + 1) / 8;
    float nom = NdotV;
    float denom = NdotV * (1.0f - k) + k;
    return nom / denom;
}

float GeometrySmith(float roughness, float NoV, float NoL) {
    float ggx2 = GeometrySchlickGGX(NoV, roughness);
    float ggx1 = GeometrySchlickGGX(NoL, roughness);
    return ggx1 * ggx2;
}
```



至此，Microfacet BRDF已经完成的介绍完，正常情况下能够得到如下的渲染结果：

![Microfacet(roughness from 0 to 1)[SIGGRAPH2017]](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/20210709114819.png)



## Kulla-Conty Approximation

在上图的渲染结果中，虽然roughness依次递增，然而直观上来看不应该由亮变暗，从数学上来看，假设入射光$L_0=1$有：

$$
\begin{equation}
E\left(\mu_{0}\right)=\int_{0}^{2 \pi} \int_{0}^{1} f\left(\mu_{0}, \mu_{i}, \phi\right) \mu_{i} d \mu_{i} d \phi \label{14}
\end{equation}
$$

根据能量守恒定律，如果Microfacet surface满足镜面反射的话，那么理论上$E(\mu_0)=1$成立，然而事实上并非如此。

因此基本思想是如何弥补这块损失的能量$1-E(\mu_0)$，`Kulla-Conty Approximation`提出在已有Microfacet BRDF的基础上加上一个能量损失BRDF$f_{ms}$，一般情况下的$f_{ms}$形式为：

$$
\begin{equation}
f_{ms}=c(1-E(\mu_i))(1-E(\mu_0)) \label{15}
\end{equation}
$$

需要满足：
$$
\begin{equation}
 E\left(\mu_{ms}\right)=\int_{0}^{2 \pi} \int_{0}^{1} f_{ms}\left(\mu_{0}, \mu_{i}, \phi\right) \mu_{i} d \mu_{i} d \phi = 1-E(\mu_0) \label{16}
\end{equation}
$$

联立方程$\eqref{15}$$\eqref{16}$解得：

$$
\begin{equation}
f_{ms}=\frac{(1-E(\mu_i))(1-E(\mu_0))}{\pi(1-E_{avg})} \label{17}
\end{equation}
$$

其中，$E_{avg}=2\int_{0}^{1}E(\mu)\mu d\mu$



## 参考资料

[^1]: https://www.graphics.cornell.edu/~bjw/microfacetbsdf.pdf
[^2]: https://learnopengl.com/PBR/Theory
[^3]: https://sites.cs.ucsb.edu/~lingqi/teaching/games202.html

