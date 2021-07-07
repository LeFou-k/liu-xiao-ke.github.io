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




在实时渲染领域，PBR表面材质主要包括Microfacet model（微表面模型）以及Disney Principled BRDFs（Most for artist），然而实际上两者并非严格意义上的**基于物理的渲染**。

本篇博客主要考虑实时渲染下的Mircofacet模型如何实现PBR，需要满足如下要求：

1. 基于Microfacet模型
2. 保持能量守恒​​
3. Physically-based BRDF

<!--more-->



