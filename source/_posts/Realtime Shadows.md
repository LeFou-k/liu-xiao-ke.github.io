---
title: Realtime Shadows
categories: 渲染
tags:
  - RTR
date: 2021/10/29
updated: 2021/10/29
mathjax: trues
---

实时阴影是真实感渲染中非常重要的一部分，在离线渲染中，我们对每个shading point进行MIS(Multiple Importance Sampling)，如果是Area Light光源，结果自然会形成十分不错的软阴影，如下图所示：

![cbox_mis rendered by Nori](https://lk-image-bed.oss-cn-beijing.aliyuncs.com/images/cbox_mis.png)

然而实际上在实时渲染的过程中，采样每道光线是一件十分奢侈的事情，即无法通过重要性采样的方法来求解渲染方程。求解渲染方程通常会利用如下不等式来近似：
$$
\begin{equation}
\int_{\Omega} f(x) g(x) \mathrm{d} x \approx \frac{\int_{\Omega} f(x) \mathrm{d} x}{\int_{\Omega} \mathrm{d} x} \cdot \int_{\Omega} g(x) \mathrm{d} x
\label{1}
\end{equation}
$$

<!--more-->

