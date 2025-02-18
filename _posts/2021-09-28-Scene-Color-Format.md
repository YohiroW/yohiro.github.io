---
title: 移动端 SceneColor 格式问题
author: Yohiro
date: 2021-09-28
categories: [Unreal, Rendering, Graphics]
tags: [graphics, rendering, profile, unrealengine]
render_with_liquid: false
img_path: /assets/images/{}/
---
## 背景

在目前帮忙的项目里，为了实现一个效果，长期在屏幕空间存在两个 Renderer，一个普通的 SceneRenderer 还有一个 SceneCaptureRenderer。

针对 SceneCaptureRenderer 做了一些裁剪的优化，目前看来没有什么问题，可是它呈现的内容少，我想让 SceneCaptureRenderer 的 SceneColorFormat 跟原本的 SceneRenderer 的 SceneColorFormat 区分开来。

当然不能直接改 SceneCapture 使用的 RT 格式，因为引擎的绘制过程中这两个 Renderer 用的是同一个 SceneColor，直接改会每帧都更新 RHI 不断地触发 Reallocating。

那就多加一个 RT 吧，可以用半分变率，内存也吃得消。于是开始调整这两个不同的 SceneColor 的格式，格式影响最大的一个是带宽的消耗，另一个是 Alpha 位数带来的问题。

## 带宽消耗

用 Snapdragon Profiler 得到了这样的带宽消耗数据：

| SceneColorFormat          | FP16 | RGB111110 | RGBA1010102 | BGRA8888 |
|:--------------------------|:-----|:----------|:------------|:---------|
| Bandwidth(R/W mega beats) | 12   | 8         | 6           | 6        |

基本上符合预期，在 Adreno 文档的 [**FAQ**](https://developer.qualcomm.com/sites/default/files/docs/adreno-gpu/snapdragon-game-toolkit/gdg/gpu/faq.html#what-is-the-performance-of-1010102-vs-111110-formats) 里也提到过：

> **What is the performance of 1010102 vs. 111110 formats?**
>
> Both formats will perform better than FP16.
> There is a hardware “fast path” for 1010102 which will allow it to perform slightly better than 111110.

这符合得到的数据，即 `BGRA8888 = RGBA1010102 > RGB111110 >> FP16`。

## 后处理与 Alpha

### Alpha

由于移动平台受限的带宽问题，大多数情况下使用 RGB111110 作为 SceneColorFormat。但是这个格式没有 Alpha 通道，而 UE4 的移动 Forward 管线中又将深度写到了 Alpha 里，导致在 Resolve 后在后处理 Pass 里拿不到深度。具体代码在 *SceneTexturesCommon.ush* 中：

``` c
/** Returns DeviceZ which is the z value stored in the depth buffer. */
float LookupDeviceZ( float2 ScreenUV )
{
    // ...

	#if MOBILE_DEFERRED_SHADING
		return Texture2DSample(MobileSceneTextures.SceneDepthTexture, MobileSceneTextures.SceneDepthTextureSampler, ScreenUV).r;
	#else
		// SceneDepth texture is not accessible during post-processing as we discard it at the end of mobile BasePass
		// instead fetch DeviceZ from SceneColor.A
		return Texture2DSample(MobileSceneTextures.SceneColorTexture, MobileSceneTextures.SceneColorTextureSampler, ScreenUV).a;
	#endif
   
    // ...
}
```

这就是为什么会有 bKeepDepthContent，通过 bKeepDepthContent 可以在后处理中访问丢掉的深度信息，从而解决没有 SceneColor.A 的问题。或者加一个 SceneDepthCopy 的 RT 用来拷贝深度，这样比较灵活。

### 后处理

在 MobileHDR 的管线下，在处理折射效果，也就是 Distortion Pass 时也是放到后处理中的，从而可以直接使用后处理 Resolve 好的 SceneColor。

然后在渲染半透明的时候，还要在 MobileBasePass 里将 BlendState 设为 RGBA，否则半透明通道无法正确写入信息。
