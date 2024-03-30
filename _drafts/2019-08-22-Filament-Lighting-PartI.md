---
title: Filament 光照篇
author: Yohiro
date: 2019-08-22
categories: [3D, Graphics]
tags: [3D, rendering, graphics, lighting]
math: true
img_path: /assets/images/Filament/
---

本篇是 [**Filament**](https://google.github.io/filament/Filament.html) 的笔记，以及部分自己的理解。

可以结合 Desktop 的渲染方式一起，看 Filament 的渲染为了更好地支持移动端，舍弃了哪些。

也可以搭配 [【GAMES101-现代计算机图形学入门-闫令琪】](https://www.bilibili.com/video/BV1X7411F744/?share_source=copy_web&vd_source=7a4dacf2c6974d860175d6d297f6d566) 食用，风味更佳。

## 光照

现在常用的引擎，如 Unity 以及 Unreal 等，对于光照的处理并不统一。

以 Unreal 为例，**点光源**亮度的单位是*流明（Lumen）*，用于表示的单位。但是**方向光**的亮度是一个没有具体单位的量，为了使点光源与 5000 lumen 的亮度相匹配，方向光的亮度需要调为 10。这种单位上的不统一，使得在场景中打光变得很不方便，也很难保持视觉一致性。

我们可以选择一个单位来统一光照，但又不方便迁移。例如在室外以亮度为 10 的方向光作为太阳光的亮度，所有其他的灯光都调整为以此为基准的相对亮度，在移动这些灯光到室内时，又会使得室内环境过亮。

这里先定义两种类型的光照：

直接光照
: 点光源、区域光和 photometric lights（光度学光源？）

间接光照
: 基于图像的照明(**I**mage **B**ased **L**ighting)

### 单位

| 光度学术语名 | 符号 | 单位 |
| 光功率 (Luminous power) | $\Phi$ | 流明 Lumen($lm$) |
| 光强 (Luminous intensity) | $I$  | 坎德拉 Candela ($cd$) or $lm/sr$ |
| 光照度 (Illuminance) | $E$ | 勒克斯 Lux ($lx$) or $lm/m^2$ |
| 光亮度/发光率 (Luminance) | $L$ | 尼特 Nit ($nt$) or $cd/m^2$ |
| 辐射功率 (Radiant power) | $\Phi_e$ | Watt ($W$) |
| 发光效能 (Luminous efficacy) | $\eta$ | Lumens per watt ($lm/W$) |
| 发光效率 (Luminous efficiency) | $V$ | Percentage (%) |

这里是 Filament 中的灯光类型，与常用的游戏引擎不太一样，

| 灯光类型 | 单位 |
| 方向光 | |
| 点光源 | |
| 聚光灯 | |
| 光度灯光 | |
| 光度遮罩灯光 | |
| 区域光 | |
| 基于图像的灯光 | |

#### 关于辐射功率单位

艺术家可能习惯于通过功率来衡量灯光的亮度，因此我们应该允许用户使用功率单位来定义灯光的亮度。转换如下:

$$\begin{equation}
\Phi = \Phi_e \eta
\end{equation}$$

这个等式中 $\eta$ 指的是发光效能，单位是流明每瓦 ($lm/W$) ，根据[维基百科中的说法](https://zh.wikipedia.org/wiki/%E7%99%BC%E5%85%89%E6%95%88%E8%83%BD)，已知最大的发光效能为 683 $lm/W$ ，那么用发光效率 $V$ 来表示的话，就是这样：

$$\begin{equation}
\Phi = \Phi_e 683 \times V
\end{equation}$$

使用各种类型灯的发光效率或发光效率将瓦特转换为流明

| 光源类型 | 发光效能($lm/W$) | 发光效率 (%) |
| 白炽灯 | 14-35 | 2-5% |
| LED 灯 | 28-100 | 4-15% |
| 荧光灯 | 60-100 | 9-15% |

$$\begin{equation}\label{derivedLuminousIntensity}
I = E \cdot d^2
\end{equation}$$

### 直接光照

上面仅对不同的光源类型定义了单位，Filament 使用物理光单位，亮度的计算是在 shader 中进行的，因此所有的光照函数都要计算亮度 $L_{out}$（也称*出射辐亮度*），该亮度值取决于光照度 $E$ 和 BSDF $f(v,l)$

$$\begin{equation}\label{luminanceEquation}
L_{out} = f(v,l)E
\end{equation}$$

#### 方向光

方向光主要用于户外环境，用来模拟阳光/月光，并不存在于真实的世界中，但因为太阳/月亮的距离足够远，所以可以近似为平行的方向光。

这种光照对表面的漫反射效果不错，但对于镜面反射的效果并不好。在寒霜引擎中，将来自太阳的方向光模拟为圆盘状的区域光，作为镜面反射的补偿。

Filament 中方向光的单位是照度($lx$)，有部分原因就是可以测量天空、阳光的具体照度值，从而简化 $L_{out}$ 的计算。

$$\begin{equation}
L_{out} = f(v,l) E_{\bot} \left< N \cdot L \right>
\end{equation}$$

其中 $E_{\bot}$ 是光源对垂直于光源的表面的照度，下面是参考值天空和阳光的参考值（满月为 1$lx$）：

| 光源 | 上午 10 点 | 中午 12 点 | 下午 5:30 |
| $Sky_{\bot}$ + $Sun_{\bot}$ | 120,000 | 130,000 | 90,000 |
| $Sky_{\bot}$ | 20,000 | 25,000 | 9,000 |
| $Sun_{\bot}$ | 100,000 | 105,000 | 81,000 |
_加州 3 月的晴朗天气下测得的照度值_

Filament 中方向光的实现:

```glsl
vec3 l = normalize(-lightDirection);
float NoL = clamp(dot(n, l), 0.0, 1.0);

// lightIntensity is the illuminance
// at perpendicular incidence in lux
float illuminance = lightIntensity * NoL;
vec3 luminance = BSDF(v, l) * illuminance;
```

#### 精准光

一般的渲染引擎中支持两种精准光，点光源和聚光灯，精准光是指*从某一位置发光的光源*，这种光源的光线均匀地照射到各个方向。但它不是物理准确的，因为下面两个原因：

1. 过于准确，且趋于无限小
2. 不遵循[平方反比准则](https://en.wikipedia.org/wiki/Inverse-square_law)

对于 1，可以引入区域光解决这一问题，但精准光的成本也较低，所以是可以接受的

$$\begin{equation}
E = L_{in} \left< N \cdot L \right> = \frac{I}{d^2} \left< N \codt L \right>
\end{equation}$$

##### 点光源

点光源仅由空间中的位置来定义

点光源的单位是发光功率（$lm$），发光功率是通过光源立体角上的发光强度的积分计算的：

$$\begin{equation}
\Phi = \int_{\Omega} I dl = \int_{0}^{2\pi} \int_{0}^{\pi} I d\theta d\phi = 4 \pi I \\
I = \frac{\Phi}{4 \pi}
\end{equation}$$

得到了发光功率，就可以很容易地计算出光照强度：

$$\begin{equation}
I = \frac{\Phi}{4 \pi}
\end{equation}$$

### IBL

### 静态光照

### 透明/半透明光照

### 遮挡

### 法线映射

## 体积效果

## 抗锯齿

## 后处理

## 参考

- [Specular BRDF Reference](http://graphicrants.blogspot.com/2013/08/specular-brdf-reference.html)
- [physically-based-shading-on-mobile](https://www.unrealengine.com/en/blog/physically-based-shading-on-mobile)
- [Game Developer Magazine Floating Point](https://randomascii.wordpress.com/2012/09/09/game-developer-magazine-floating-point/)
- [Moving Frostbite to PBR](https://media.contentapi.ea.com/content/dam/eacom/frostbite/files/s2014-pbs-frostbite-slides.pdf)

[^Disney]: [Physically Based Shading at Disney](https://media.disneyanimation.com/uploads/production/publication_asset/48/asset/s2012_pbs_disney_brdf_notes_v3.pdf)
[^Heitz14]: [Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs](https://www.jcgt.org/published/0003/02/03/paper.pdf)