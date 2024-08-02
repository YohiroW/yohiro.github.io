---
title: 霍尔顿序列
author: Yohiro
date: 2023-12-03
categories: []
tags: []
render_with_liquid: false
img_path: /assets/images/{}/
---
## 背景

```cpp
FVector3f VolumetricFogTemporalRandom(uint32 FrameNumber)
{
    // Center of the voxel
    FVector3f RandomOffsetValue(.5f, .5f, .5f);

    if (GVolumetricFogJitter && GVolumetricFogTemporalReprojection)
    {
        RandomOffsetValue = FVector3f(Halton(FrameNumber & 1023, 2), Halton(FrameNumber & 1023, 3), Halton(FrameNumber & 1023, 5));
    }

    return RandomOffsetValue;
}
```

## 实现

```c
/** [ Halton 1964, "Radical-inverse quasi-random point sequence" ] */
float Halton(uint Index, uint Base)
{
    float r = 0.0;
    float f = 1.0;

    float BaseInv = 1.0 / Base;
    while (Index > 0)
    {
        f *= BaseInv;
        r += f * (Index % Base);
        Index /= Base;
    }

    return r;
}
```

## 参考

- [Halton序列](https://en.wikipedia.org/wiki/Halton_sequence)
- [低差异序列 (low-discrepancy sequences)之Halton序列均匀产生多维随机数的介绍与实现](https://blog.csdn.net/lr_shadow/article/details/120455297)