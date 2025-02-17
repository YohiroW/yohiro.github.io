---
title: Filament 光照篇
author: Yohiro
date: 2019-08-22
categories: [3D, Graphics]
tags: [3D, rendering, graphics, lighting]
math: true
media_subpath: /assets/images/Filament/
---

本篇是 [**Filament**](https://google.github.io/filament/Filament.html) 的笔记，以及部分自己的理解。

可以结合 Desktop 的渲染方式一起，看 Filament 的渲染为了更好地支持移动端，舍弃了哪些。

也可以搭配 [【GAMES101-现代计算机图形学入门-闫令琪】](https://www.bilibili.com/video/BV1X7411F744/?share_source=copy_web&vd_source=7a4dacf2c6974d860175d6d297f6d566) 食用，风味更佳。

# 光照

现在常用的引擎，如 Unity 以及 Unreal 等，对于光照的处理并不统一。

以 Unreal 为例，**点光源**亮度的单位是*流明（Lumen）*，用于表示的单位。但是**方向光**的亮度是一个没有具体单位的量，为了使点光源与 5000 lumen 的亮度相匹配，方向光的亮度需要调为 10。这种单位上的不统一，使得在场景中打光变得很不方便，也很难保持视觉一致性。

我们可以选择一个单位来统一光照，但又不方便迁移。例如在室外以亮度为 10 的方向光作为太阳光的亮度，所有其他的灯光都调整为以此为基准的相对亮度，在移动这些灯光到室内时，又会使得室内环境过亮。

这里先定义两种类型的光照：

直接光照
: 点光源、区域光和 photometric lights（光度学光源？）

间接光照
: 基于图像的照明(**I**mage **B**ased **L**ighting)

## 单位

| 光度学术语名 | 符号 | 单位 |
| 光功率 (Luminous power) | $\Phi$ | 流明 Lumen($lm$) |
| 光强 (Luminous intensity) | $I$  | 坎德拉 Candela ($cd$) or $lm/sr$ |
| 光照度 (Illuminance) | $E$ | 勒克斯 Lux ($lx$) or $lm/m^2$ |
| 光亮度/发光率 (Luminance) | $L$ | 尼特 Nit ($nt$) or $cd/m^2$ |
| 辐射功率 (Radiant power) | $\Phi_e$ | Watt ($W$) |
| 发光效能 (Luminous efficacy) | $\eta$ | Lumens per watt ($lm/W$) |
| 发光效率 (Luminous efficiency) | $V$ | Percentage (%) |

下面是 Filament 中的灯光类型以及单位。

| 灯光类型 | 单位 |
| 方向光 | 照度 Lux ($lx$) or $lm/m^2$ |
| 点光源 | 光功率 Lumen ($lm$) |
| 聚光灯 | 光功率 Lumen ($lm$) |
| 光度灯光 | 光强 Candela ($cd$) or $lm/sr$ |
| 光度遮罩灯光 | 光功率 Lumen ($lm$) |
| 区域光 | 光功率 Lumen ($lm$) |
| 基于图像的灯光 | 光亮度/发光率 Nit ($nt$) or $cd/m^2$ |

### 关于辐射功率单位

艺术家可能习惯于通过功率来衡量灯光的亮度，因此我们应该允许用户使用功率单位来定义灯光的亮度。转换如下:

$$\begin{equation}
\Phi = \Phi_e \eta
\end{equation}$$

这个等式中 $\eta$ 指的是发光效能，单位是流明每瓦 ($lm/W$) ，根据[维基百科中的说法](https://zh.wikipedia.org/wiki/%E7%99%BC%E5%85%89%E6%95%88%E8%83%BD)，已知最大的发光效能为 **683** $lm/W$ ，那么用发光效率 $V$ 来表示的话，就是这样：

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

## 直接光照

上面仅对不同的光源类型定义了单位，Filament 使用物理光单位，亮度的计算是在 shader 中进行的，因此所有的光照函数都要计算亮度 $L_{out}$（也称*出射辐亮度*），该亮度值取决于光照度 $E$ 和 BSDF $f(v,l)$

$$\begin{equation}\label{luminanceEquation}
L_{out} = f(v,l)E
\end{equation}$$

### 方向光

方向光主要用于户外环境，用来模拟阳光/月光，并不存在于真实的世界中，但因为太阳/月亮的距离足够远，所以可以近似为平行的方向光。

![](diagram_directional_light.png)

这种光照对表面的漫反射效果不错，但对于镜面反射的效果并不好。在寒霜引擎中，将来自太阳的方向光模拟为圆盘状的区域光，作为镜面反射的补偿。

Filament 中方向光的单位是照度($lx$)，有部分原因就是可以测量天空、阳光的具体照度值，从而简化 $L_{out}$ 的计算。

$$\begin{equation}
L_{out} = f(v,l) E_{\bot} \left< N \cdot L \right>
\end{equation}$$

其中 $E_{\bot}$ 是光源对垂直于光源的表面的照度，下面是加州 3 月的晴朗天气下测得的天空和阳光的照度值（满月为 1$lx$）：

| 光源 | 上午 10 点 | 中午 12 点 | 下午 5:30 |
| $Sky_{\bot}$ + $Sun_{\bot}$ | 120,000 | 130,000 | 90,000 |
| $Sky_{\bot}$ | 20,000 | 25,000 | 9,000 |
| $Sun_{\bot}$ | 100,000 | 105,000 | 81,000 |

方向光的实现:

```glsl
vec3 l = normalize(-lightDirection);
float NoL = clamp(dot(n, l), 0.0, 1.0);

// lightIntensity is the illuminance
// at perpendicular incidence in lux
float illuminance = lightIntensity * NoL;
vec3 luminance = BSDF(v, l) * illuminance;
```

### 精准光

一般的渲染引擎中支持两种精准光，点光源和聚光灯，精准光是指*从某一位置发光的光源*，这种光源的光线均匀地照射到各个方向。但它不是物理准确的，因为下面两个原因：

1. 过于准确，且趋于无限小
2. 不遵循[平方反比原则](https://en.wikipedia.org/wiki/Inverse-square_law)

对于 1，可以引入区域光解决这一问题。对于 2 直接引入材质表面着色点到光源的距离 $d$ 来解决，如下。

$$\begin{equation}
E = L_{in} \left< N \cdot L \right> = \frac{I}{d^2} \left< N \cdot L \right>
\end{equation}$$

#### 点光源

点光源仅由空间中的位置来定义。

![](diagram_point_light.png)

点光源的单位是发光功率（$lm$），发光功率是通过光源立体角上的发光强度的积分计算的：

$$\begin{equation}
\Phi = \int_{\Omega} I dl = \int_{0}^{2\pi} \int_{0}^{\pi} I d\theta d\phi = 4 \pi I \\
I = \frac{\Phi}{4 \pi}
\end{equation}$$

得到了发光功率，就可以很容易地计算出光照强度：

$$\begin{equation}
I = \frac{\Phi}{4 \pi}
\end{equation}$$

结合上面提到的 $L_{out}=f(v,l)E$，最终可以得到：

$$\begin{equation}
L_{out} = f(v,l) \frac{\Phi}{4 \pi d^2} \left< N \cdot L \right>
\end{equation}$$

#### 聚光灯

聚光灯由位置、方向和两个锥角 $$\theta_{inner}$$ 和 $$\theta_{outer}$$ 来定义。一般的光照使用 $$\frac{1}{d^2}$$ 来定义距离的衰减，而对于聚光灯 $$\theta_{inner}$$ 和 $$\theta_{outer}$$ 定义了聚光灯锥形区域两侧的角度衰减。

![](diagram_spot_light.png)

$$\begin{equation}\label{spotLightLuminousPower}
\Phi = \int_{\Omega} I dl = \int_{0}^{2\pi} \int_{0}^{\theta_{outer}} I d\theta d\phi = 2 \pi (1 - cos\frac{\theta_{outer}}{2})I\\\\
I = \frac{\Phi}{2 \pi (1 - cos\frac{\theta_{outer}}{2})}
\end{equation}$$

但在这里，聚光灯的圆锥与亮度耦合在一起，调整锥角的角度时，也会同时影响亮度。因此，需要将角度和亮度解耦以便灯光师调节：

$$\begin{equation}\label{spotLightLuminousPowerB}
\Phi = \pi I \\\\
I = \frac{\Phi}{\pi} \\\\
\end{equation}$$

聚光灯的两种评估方式：

从吸收光的角度
:

$$\begin{equation}\label{spotAbsorber}
L_{out} = f(v,l) \frac{\Phi}{\pi d^2} \left< N \cdot L \right> \lambda(l)
\end{equation}$$

从反射光的角度
:

$$\begin{equation}\label{spotReflector}
L_{out} = f(v,l) \frac{\Phi}{2 \pi (1 - cos\frac{\theta_{outer}}{2}) d^2} \left< N \cdot L \right> \lambda(l)
\end{equation}$$

以上两式中的 $$\lambda(l)$$ 项是聚光灯的角度衰减因子，描述如下：

$$\begin{equation}\label{spotAngleAtt}
\lambda(l) = \frac{l \times spotDirection - cos\theta_{outer}}{cos\theta_{inner} - cos\theta_{outer}}
\end{equation}$$

#### 衰减函数

光照的衰减需要正确地评估衰减因子，简单的 $\frac{1}{d^2}$ 项并不好使，因为：

1. 当几何体与光源相接触，可能会有 $d$ = 0，从而导致除以 0。
2. 理论上每个光源的范围是无限远的，在有多光源的场合会导致 $\frac{1}{d^2}$ 永远不会为 0，如果要正确着色，需要对场景内每一个光源进行评估。

为解决上述问题 1，Filament 的解决方式是将点光源假设为一个小半径为 1 cm 的球形区域光。

$$\begin{equation}\label{finitePunctualLight}
E = \frac{I}{max(d^2, {0.01}^2)}
\end{equation}$$

为解决问题 2， 可以为每个光源引入一个影响半径的参数解决。通过引入这一变量可以展示给美术光源所影响到的范围，也能方便引擎渲染时剔除光源。

由 GLSL 实现的基于物理的精准光源参考：
```glsl
float getSquareFalloffAttenuation(vec3 posToLight, float lightInvRadius)
{
    float distanceSquare = dot(posToLight, posToLight);
    float factor = distanceSquare * lightInvRadius * lightInvRadius;
    float smoothFactor = max(1.0 - factor * factor, 0.0);
    return (smoothFactor * smoothFactor) / max(distanceSquare, 1e-4);
}

float getSpotAngleAttenuation(vec3 l, vec3 lightDir, float innerAngle, float outerAngle)
{
    // 缩放和偏移计算可以在CPU端完成
    float cosOuter = cos(outerAngle);
    float spotScale = 1.0 / max(cos(innerAngle) - cosOuter, 1e-4)
    float spotOffset = -cosOuter * spotScale

    float cd = dot(normalize(-lightDir), l);
    float attenuation = clamp(cd * spotScale + spotOffset, 0.0, 1.0);
    return attenuation * attenuation;
}

vec3 evaluatePunctualLight()
{
    vec3 l = normalize(posToLight);
    float NoL = clamp(dot(n, l), 0.0, 1.0);
    vec3 posToLight = lightPosition - worldPosition;

    float attenuation;
    attenuation  = getSquareFalloffAttenuation(posToLight, lightInvRadius);
    attenuation *= getSpotAngleAttenuation(l, lightDir, innerAngle, outerAngle);

    // 这里的光照强度的单位是坎德拉，可由光照功率转换而来
    vec3 luminance = (BSDF(v, l) * lightIntensity * attenuation * NoL) * lightColor;
    return luminance;
}
```
以上的部分计算可在 CPU 端完成。

### 光度学光源

TODO

### 面光源

TODO

### 光源参数

Filement 的目标是使光源参数易于美工和开发人员使用. 为此需要直观方便，因此光源颜色(或色调)与光源强度相互独立，且光源的颜色定义为线性 RGB 颜色(或在外部工具/UI 中使用 sRGB)。

| 参数             | 定义            |
|:----------------|:----------------|
| 光源类型         | 平行光 Directional, 点光源 point, 聚光灯 spot, 面光源 area |
| 方向             | 平行光、点光、聚光灯、面光的方向 |
| 光源颜色         | 发射出的光的颜色，指定为线性 RGB，在外部工具中可以 sRGB 或色温指定 |
| 光照强度         | 直观的反映为灯光的亮度，具体单位取决于光源类型      |
| 衰减半径         | 最大的影响距离     |
| 内锥角           | 聚光灯形成的内圆锥体的角度   |
| 外锥角           | 聚光灯形成的外圆锥体的角度   |
| 长度            | 面光源的长度, 用于创建线性或管状灯光 |
| 半径            | 面光源的半径, 用于创建球形或管状灯光 |

#### 色温

真实世界中的人造灯光通常由它们的色温来定义, 以开尔文(K)为单位. 光源的色温是理想黑体辐射体的温度, 此黑体辐射出的光的色调与光源相当. 为方便起见, 这些工具应该支持美工使用色温指定光源的色调(有意义的范围为1,000 K到12,500 K)。

### 预曝光

## IBL

## 静态光照

## 透明/半透明光照

## 遮挡

## 法线映射

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