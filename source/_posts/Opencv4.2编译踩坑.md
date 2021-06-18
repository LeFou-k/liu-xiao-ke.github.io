---
title: Opencv编译踩坑
categories: 配置
tags:
  - Opencv
mathjax: false
---

# Opencv编译踩坑

## OpenCV4.2

主要参考：https://gist.github.com/raulqf/f42c718a658cddc16f9df07ecc627be7

注意几点：

1. CUDA_ARCH_BIN的版本详见[Nvidia](https://developer.nvidia.com/cuda-gpus)官网，查看对应GPU的版本即可

2. 出现"OpenCV not found"之类的错误，添加如下cmake代码

   ```cmake
   set(OpenCV_DIR ~/opencv/build/bin) #自己build完的目录
   find(Opencv Required 4)
   ```

3. 编译opencv程序出现"include<opencv2>not found"的错误，添加如下代码：

   ```cmake
   set(OpenCV_INCLUDE_DIR /usr/local/include/opencv4) #一般是这个地址
   include_directories($OpenCV_INCLUDE_DIR)
   ```

<!--more-->

## Opencv3.2.0

出现“cmake error:CUDA_nppi_LIBRARY (ADVANCED)”后，需要设置：

```bash
cmake -D CUDA_nppi_LIBRARY=true ..
```

出现：

```bash
for file: [/home/yk/opencv-3.2.0/3rdparty/ippicv/downloads/linux-808b791a6eac9ed78d32a7666804320e/ippicv_linux_20151201.tgz]
      expected hash: [808b791a6eac9ed78d32a7666804320e]
        actual hash: [b0f455bf2adcf42d2217a76d51b2d165]
             status: [28;"Timeout was reached"]
```

需要手动下载[ippicv](https://raw.githubusercontent.com/Itseez/opencv_3rdparty/81a676001ca8075ada498583e4166079e5744668/ippicv/ippicv_linux_20151201.tgz)再覆盖**opencv3.2/3rdparty/ippicv/downloads/linux-***下的同名文件重新cmake即可。


