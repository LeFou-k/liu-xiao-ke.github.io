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

PCSS即采用动态调整滤波核大小的PCF，滤波核越大阴影越soft，反之越hard，直觉上滤波核的大小应该和阴影到Blocker的距离相关，严格的证明如下所示：

<img src="https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/Penumbra size.png" alt="image-20211102135958189" style="zoom:67%;" /> 

如图所示，根据相似三角形有：
$$
\begin{equation}
w_{\text {Penumbra }}=\frac{\left(d_{\text {Receiver }}-d_{\text {Blocker }}\right) \cdot w_{\text {Light }}}{d_{\text {Blocker }}}
\label{4}
\end{equation}
$$
从$\eqref{4}$式中得知，此时我们直接查shadingpoint在shadowmap上的映射可知$d_{Receiver}$，$w_{Light}$是已知参数，唯一未知项即$d_{Blocker}$，如果$d_{Blocker}$可知那么很自然的可以得知$w_{Penumbra}$亦即PCSS中半影滤波核的大小。

总结下来PCSS整体分为两步：第一步求解$d_{Blocker}$，第二步根据$d_{Blocker}$得到滤波核大小再进行滤波。

具体来看，

1. Blocker Search(getting the average **blocker** depth in a **certain region**)

   注意Blocker Search的搜索范围，仍然是一个相似三角形的做法，如下图：

   ![image-20211102142101083](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/Block Search Centain Region.png)

   求得搜索范围后即可在区间内搜索求解Average blocker depth，伪代码如下所示：

   ```glsl
   //get average blocker depth
   float findBlocker(){
       float blockSize = LIGHT_WIDTH * (zReceiver - zNear) / zReceiver;
       float blockerNum = 0.0, avg = 0.0;
       for(int i = 0; i < BLOCKER_SEARCH_SAMPLES; ++i){
           vec2 sampleCoord; //sample a coord in blockSize range
           float sampledepth; //get sample depth
           if(sampledepth < zReceiver){
               avg += sampledepth;
               blockerNum += 1.0;
           }
       }
       return avg / blockerNum;
   }
   ```

   

2. Penumbra estimation(use average blocker depth to estimate filter size)

   这一步就相对很容易了，直接套用$\eqref{4}$式公式求解得到PCF的滤波核大小。

3. Percentage Closer Filtering

   参考PCF方法，已知$w_{Penumbra}$可直接按照PCF流程滤波求解即可。

最后附上PCSS的结果，可以很明显的看到阴影随着到Blocker距离的增大边缘也逐渐虚化，PCSS可以认为是一种最准确的结果，后续的VSSM和MSM也只是尽可能的加速这个求解的过程。

![PCSS_xxx](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/PCSS.png)

### VSSM

VSSM(Variance Soft Shadow Mapping)，让我们回到最开始软阴影原理部分，无论是PCF还是PCSS的目的都是求Shading Point周围点的Visibility情况，即“比当前点更visible占总体的比例”，进一步这是一个$P\{x > t\}$的问题，由此可以想到，如果能够知道周围点depth分布的话，那么就可以估计出$P\{x>t\}$的近似值了。

VSSM就是这样一种想法，通过快速计算区域内的均值和方差从而近似求解。在VSSM中，甚至可以不需要正态分布，仅通过切比雪夫不等式得到对$P\{x>t\}$的近似估计，见下式：
$$
\begin{equation}
P(x>t) \leq \frac{\sigma^{2}}{\sigma^{2}+(t-\mu)^{2}}
\label{5}
\end{equation}
$$
同样，要计算$P\{x>t\}$的结果只需得到整体的$\mu$和$\sigma$即可。

VSSM算法由于避免了搜索过程，因此可以大大提高PCSS中第一步Blocker Search和第三步PCF的速度，下面具体来看VSSM的实现方式。

#### Blocker Search

回忆一下Blocker Search的执行过程，通过搜索相应的Blocker size得到Average Blocker depth，那么如何加速这个过程呢？分析Search区间可知，对当前Shading point只有Blocker和非Blocker两种像素，以$Z_{occ}$和$Z_{unocc}$来分别表示，以$N_o$和$N_u$来表示搜索区间Blocker和非Blocker的数目，$Z_{avg}$表示搜索区间深度的均值，则显然有下式：
$$
\begin{equation}
\frac{N_{u}}{N} z_{\text {unocc }}+\frac{N_{o}}{N} z_{o c c}=z_{A v g}
\end{equation}
$$
以及：
$$
\begin{equation}
N_{u} + N_{o} = N
\end{equation}
$$
已知$z_{avg}$可快速估计（见下节），$\frac{N_o}{N}$可通过$P\{x>t\}$估计，$z_{occ}$为所求结果，VSSM大胆估计$z_{unocc}=t$，即可求得$z_{occc}$



#### $\mu$和$\sigma$的快速估计

由均值和方差公式知，
$$
\begin{equation}
\operatorname{Var}(X)=\mathrm{E}\left(X^{2}\right)-\mathrm{E}^{2}(X)
\end{equation}
$$
因此求解$X^{2}$的均值和$X$的均值即可，有**MIPMAP**和**SAT**两种方法。

1. MIPMAP

   如下图所示，可根据ShadowMap中的depth构建MipMap，之后根据Blocker Search size找到对应层进行双向插值，如果是两层之间则三向插值即可，$X^2$求解同理。

   ![image-20211102154150650](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/MIPMAP.png)

2. SAT(Summed Area Table)

   SAT的核心思想即维护一个表记录从0位置到当前位置的总和，在生成ShadowMap的过程中额外生成一张$depth$和$depth^2$的表即可。

   ```c++
   vector<vector<float>> SAT;
   SAT[i][j];//means sum up [0][0] to [i][j]
   //calculate (p,q) to (m,n)
   float avg = (SAT[m][n]-SAT[m][q]-SAT[p][n]+SAT[p-1][q-1]) / (m - p) / (n - q);
   ```

   ![image-20211102154707593](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/SAT.png)



VSSM整体流程如下：

1. 确定Blocker Search的范围
2. 根据MipMap或SAT求解Blocker Search范围内的$\mu$和$\sigma$
3. 根据Blocker Search优化方法估计$Z_{occu}$
4. 由$Z_{occu}$计算得到PCF的filter size
5. 求PCF filter范围内的$\mu$，$\sigma$
6. 根据式$\eqref{5}$切比雪夫不等式求得visibility

#### Artifacts

直接上结论，VSSM会导致light leaking的现象，根本原因在于式$\eqref{5}$切比雪夫不等式的适用范围，切比雪夫不等式仅在$t >Z_{avg}$时有效，即如果$Z_{avg}$比较大（Overly bright）会出现这种情况：

![VSSM's artifacts](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/Light Leaking.png)

解决办法即下面的MSM(Moment Shadow Mapping)

### MSM

MSM采用了更加准确的方式来拟合depth 分布从而达到更加精确的效果，Moments即$x, x^{2}, x^{3}, x^{4}, \ldots$，实际VSSM只采用了前两项，实际上4已经能够足够好的你和depth的CDF(Culmative Distribution Function)累积分布函数，实现方法类似VSSM，在求ShadowMap时额外计算$z, z^{2}, z^{3}, z^{4}$，相比于VSSM能够得到更加精确的结果，但是显然空间更加costly

![VSSM vs MSM](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/MSM vs VSSM.png)



## Conclusion

Realtime shadows大部分相关算法都已经介绍完毕，主要围绕PCSS的引出以及如何优化PCSS过程展开，基本基于GAMES202加上《Real time rendering》的一些内容，可以视为GAMES202的课程笔记。



## 参考资料

[^1]: 《Realtime Rendering 4th edition》
[^2]: GAMES202 Realtime Shadows

