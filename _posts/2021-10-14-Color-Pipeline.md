---
title: 颜色管线与色彩空间
author: Yohiro
date: 2021-10-14
categories: [Rendering]
tags: [engine, unrealengine, colorspace, HDR]
math: true
render_with_liquid: false
img_path: /assets/images/ColorPipeline/
---

在处理材质贴图、Sampler 或 RenderTarget 时，经常会看到 sRGB/Linear 字样。颜色在渲染时是如何编码解码的？在不同的阶段是怎样的颜色类型？发生怎样的转换？这篇文章尝试回答这些问题。

## 概念

- **颜色空间（Color Space）**

  生成红色、绿色、蓝色和白色的具体颜色。由于可以定义多个颜色空间，这会导致 1,0,0（为例）在一个颜色空间中的颜色更为饱和，例如：`Rec709`、`Rec2020`、`sRGB`、`ACES 2065-1`等。

- **颜色编码（Color Encoding）**

  颜色的数字呈现方式，以及该呈现方式所产生的光，例如：线性、sRGB、伽马、PQ（ST 2084）和多种供应商专用的日志编码。

- **伽马值（Gamma）**

  一种非线性的颜色编码方式，主要用于对亮度值进行编码，使用的表达式为$Output=A\times{Input}^{Gamma}$，A 通常使用常数 1.0，这里的`Gamma`就是我们所说的伽马校正值。

- **色域（Gamut）**

  可通过颜色空间、输出设备或特定图像在某一时间的颜色集所代表的颜色子集。

- **线性颜色空间（Linear Color Space）**

  一种使用了线性颜色编码方式的颜色空间，它所表示的光照量与存储值成比例，因此该数值翻倍将会使亮度翻倍。

- **sRGB（sRGB）**

  在计算机图像（和虚幻引擎）中，sRGB 指的是一种特定的颜色空间和颜色编码方式。你可以使用线性 sRGB，或者 sRGB 编码的 sRGB。如果不澄清颜色空间或颜色编码方式的背景，该术语很可能产生混淆。

## 简化的渲染管线

![简化的渲染管线图](rtaImage.png)

渲染通常分为四个阶段：
1. 获取纹理输入，将其导入 LinearColor 的颜色空间中
2. 确定图元可见性和着色，将结果输入 SceneColor
3. 对 SceneColor 进行后处理
4. 最后输出到视口设备中显示

## 颜色转换

![](rtaImage (1).png)

虚幻引擎的隐式工作空间是线性 sRGB。纹理的编码在经过转化后，会将纹理导入到工作空间中。目前这部分是由纹理的 sRGB 勾选框处理的，它表明了纹理文件是否为线性（标记为 false），或者文件是否使用 sRGB 编码（标记为 true）。
渲染和后期处理都会在线性空间中直接进行，直到抵达 ToneCurve（ToneMapper 的一部分）。色调曲线会将较大的场景动态范围压缩到 0.0-1.0 之间。
最后，我们会从 Linear 转换回 sRGB 编码像素进行展示。

## sRGB 到 Linear 的转换

假设我们有一张 sRGB 编码的图片，如下所示，灰度为 [0,255]
![](rtaImage (2).png)

读取数值时，它会通过`sRGBToLinear`函数进行转换，从而降低了这部分数值。
![](rtaImage (3).png)

这意味输入值是 128（或 0.5）时，该值会变为线性值的 0.214。
![](rtaImage (4).png)

这样我们就能使用较低比特的输入来代表暗色值。当数据转换为线性之后，就需要更多比特来代表线性值。我们在场景渲染时会使用 16 位浮点，以便同时代表极小的暗色值和极大的亮色值。
![](rtaImage (5).png)

## 扩展的渲染管线

### RGB 空间下渲染为 Linear

![](rtaImage (6).png)
![](rtaImage (7).png)
![](rtaImage (8).png)
![](rtaImage (9).png)

### HDR

![](rtaImage (10).png)

## 参考

- [颜色管线基础](https://udn.unrealengine.com/s/article/Color-Pipeline-Basics)
- [High Dynamic Range Display Output](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/HDRDisplayOutput/)
- [在高/标准动态范围显示器上将 DirectX 与高级颜色配合使用](https://learn.microsoft.com/zh-cn/windows/win32/direct3darticles/high-dynamic-range#option-2-use-uint10rgb10-pixel-format-and-hdr10bt2100-color-space)