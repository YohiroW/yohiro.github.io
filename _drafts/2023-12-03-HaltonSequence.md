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

## 应用

低维度的素数中具有不错的表现，但在稍大的素数生成的序列间存在相关性问题，比如 17、19。为避免这种情况，通常会删除前 20 个条目。还有其他的解决方案，详见[百科](https://en.wikipedia.org/wiki/Halton_sequence)。

在游戏领域，绝大多数情况下使用前四个素数：2，3，5，7 就够用了。

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

## 扩展

霍尔顿序列的索引可以是[莫顿码](https://en.wikipedia.org/wiki/Z-order_curve)，比如在 volumetric light map 的随机采样中：

```c
    int3 CellPosInVLM = (BrickRequests[BrickIndex].xyz * 4 + CellPosInBrick) * BrickRequests[BrickIndex].w;

    float3 RandSample = float3(
        Halton(MortonEncode3(CellPosInVLM) + EffectiveRenderPassIndex, 2), 
        Halton(MortonEncode3(CellPosInVLM) + EffectiveRenderPassIndex, 3),
        Halton(MortonEncode3(CellPosInVLM) + EffectiveRenderPassIndex, 5)
    );
```

## 参考

- [Halton序列](https://en.wikipedia.org/wiki/Halton_sequence)
- [低差异序列 (low-discrepancy sequences)之Halton序列均匀产生多维随机数的介绍与实现](https://blog.csdn.net/lr_shadow/article/details/120455297)