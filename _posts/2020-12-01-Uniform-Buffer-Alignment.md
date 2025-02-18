---
title: Uniform Table 的偏移问题
author: Yohiro
date: 2020-12-01
categories: [Rendering]
tags: [engine, programming, rendering, opengl, unrealengine]
img_path: /assets/images/UniformTable/
---
## 背景

最近 UE4.21 的项目中遇到了一个 Bug，具体问题是这样的：

玩家可以蹲伏在草丛中进入隐匿状态，进入隐匿状态后，草丛有一个不透明度降低的效果。问题出在这种效果在有些平台存在，有些平台不存在。

后来经过抓帧、调试这样的一番探索，发现问题出在 uniform buffer 里。

## 问题

真正的问题是，在某个 uniform buffer 之后，从 NSight 中观测到的数据出现了偏移。

UE4里的 uniform buffer 定义在 SceneView 里：

```cpp
// View uniform buffer member declarations
#define VIEW_UNIFORM_BUFFER_MEMBER_TABLE \
    VIEW_UNIFORM_BUFFER_MEMBER(FMatrix, TranslatedWorldToClip) \
    VIEW_UNIFORM_BUFFER_MEMBER(FMatrix, WorldToClip) \
 
    ....
 
    VIEW_UNIFORM_BUFFER_MEMBER(FVector4, HairRenderInfo) \
    VIEW_UNIFORM_BUFFER_MEMBER(uint32, HairRenderInfoBits) \
```

UE4 里 GLSL 的 uniform buffer 默认使用的是 std140 的布局。在这个 uniform table 里，如果成员中有数组，便会产生 uniform 成员偏移的问题，而这些偏移仅出现在使用 GLSL 的平台上。

```cpp
ralloc_asprintf_append(buffer, "layout(std140) uniform %s\n{\n", block_name);
```

因为这种布局会将数组中的元素，不论类型，都当做 `16 字节` 来对齐。

官方的原话是 **The array stride (the bytes between array elements) is always rounded up to the size of a vec4 (ie: 16-bytes).**

![](Layout.png)

也就是说如果你在 uniform table 中声明了一个成员，比如：

```cpp
VIEW_UNIFORM_BUFFER_MEMBER_ARRAY(float, LightVolume[2])
```

这段代码在 HLSL 和 GLSL 中内存的布局是不同的，在 HLSL 中和我们 C++ 中声明的长度一致。而在 GLSL 中，由于 layout 是 std140，所以这里的 LightVolume[2] 实际上占用了 32 byte，而不是 8 byte。

这就使得编译 shader 时，在加入了一些 padding 后，从 CPU 传到 GPU 的值有了偏移。

## 参考

- [**Interface Block (GLSL)**](https://www.khronos.org/opengl/wiki/Interface_Block_(GLSL))