---
title: Delta Color Compression
author: Yohiro
date: 2022-08-21
categories: [Graphics,Optimization,AMD]
tags: [graphics, optimization, amd, compression]
render_with_liquid: false
img_path: /assets/images/{}/
---

## 问题

前段时间遇到一个平台相关问题，问题是这样：

在 UE 里使用了 SceneCapture 处理的一个低分辨率的 RT，经处理后要做 CopyToResolveTarget，而在这之后作为 Src 的 SceneCaptureRT 没有被拉伸到目标 RT 的大小。最后怀疑了会不会是 DX12 的 CopyTextureRegion 的问题，于是去查了相关资料，最后在一个 2020 年的 release note 里找到了这条信息：

> ...其中问题在于，当在源纹理上启用DCC时，CopyTextureRegion() 将忽略源矩形并将整个纹理复制到目标中的同一子矩形。

可以看到 CopyTextureRegion 对于 DCC 的支持不是很完善。在相应的 RT 上添加 TexCreate_DisableDCC 的 Flag 后问题确实可以解决问题，但还是想看看 DCC 究竟干了什么，为什么禁止了这个 Flag 问题可以解决。

## Delta Color Compression

> 除解压缩外，以下信息仍然适用于 AMD RDNA/RDNA2 架构。
{: .prompt-warning }

带宽是一种珍贵的资源，虽然硬件在不断进步，但是游戏渲染的分辨率越来越高，DataBuffer 越来越大，在读写 RenderTarget 时占用了很多带宽。引入 **D**elta **C**olor **C**ompression，就是为了缓解带宽的压力。

DCC 是一种针对特定区域的无损压缩，通过数据一致性来减少所需的带宽。如同传统的压缩器，只不过是用于 3D 图形渲染，DCC 的思想关键在于它是逐 block 处理而不是逐像素处理，在一个 block 之内只有一个值是全精度存储，其余所存储的是相对于全精度值的增量，如果同一个 block 内使用了近似的色彩，则增量值可以使用比输入少得多的位来存储。

DCC 的压缩器在 Color block 內部，它允许图形以类似于 DepthStencil target 的方式压缩 Color target。实际的硬件实现中会根据实际数据和访问方式来自动调节 block 的大小以优化潜在的随机访问。

通常，我们读取数据的频率比写入数据的频率高得多。为了充分发扬节省带宽这一优势，着色器核心被赋予了读取新的压缩颜色以及所有现有压缩表面的能力。这允许在 Render to texture 中完全跳过解压缩操作，在资源状态发生变换时，比如  Render target 到 Texture 的过程中，Barrier transition 是 NoOp，从而不触发昂贵的解压缩。

### 注意事项

像 EarlyZ 一样，存在一些应用场景会使该优化失效，下面列出了一些要注意的事项，更完整的可以阅读 [Getting the Most Out of Delta Color Compression](https://gpuopen.com/learn/dcc-overview/)。

- 如无必要，不要把 RenderTarget 标记为 Shader 可读的资源

    Shader 可读的资源，比如 SRV，通常没有很好的被压缩。硬件中的 Shader 单元可以直接压缩数据，但是有一些另外的压缩选项只有在由 Shader core 读取数据的行为被禁止的情况下才可以进行。这一点对 MSAA 的 Depth target 影响最大，如果它不作为 SRV，它会取得较高的压缩质量。

- 试着用 32 bit 的浮点深度格式，而不是 16 bit

    当作为 Shader 资源时，D32Fs 可能会比 D16s 被压缩的更小。它们的不同在于被分配的大小和解压时的带宽占用，然而解压本身并不是一个非常频繁的操作，除非在屏幕上很小的区域中存在很密集的 Mesh，这些 Mesh 具有很细小的三角形。D32Fs 也允许使用 Reverse-z 来添加近裁切面的精度，忽略掉解压地场景，D32Fs 对比 D16Fs 几乎没有消耗。

    在 GCN 架构的 AMD GPU 上，不存在真正的 24 bit 的 Depth target，24 位的深度会被当作忽略了 8 位精度的 32 位的深度来处理，因此 24 位到 32 位的切换几乎没有额外消耗。

- 尽量写所有的颜色通道

    对于非压缩的数据，执行各通道独立的写入没有问题。但对压缩的数据，数据需要先被读取，解压，更新，最后再写回原先的通道中。为了更有效地压缩，最好同时完全覆写所有通道。

不过这些注意事项也不是必定要遵守的，还是应该根据工程的实际情况来进行判断。

由于目前不是所有的 GPU 单元都可以读写压缩数据，所以有一些场景仍会导致 DCC 失效，从而触发解压缩，其中针对 [Copy engine](https://learn.microsoft.com/zh-cn/windows/win32/direct3d12/user-mode-heap-synchronization) 的有这么两条：

> - Generally once flagged as a copy source or target, we decompress first because we don’t know where it is going.
> - Raw copies can be done between surfaces of **the same type and size**… if the driver knows that this is going to happen.

回到文章一开始所说的 bug，CopyTextureRegion 会始终把具有 DCC 标记的渲染资源当作与目标大小一致的 RenderTarget 来 resolve。这么做是可能为了确保 DCC 起效，因为如果启用了 DCC 的源纹理与目标纹理不一致，那么源纹理无法经由 DCC 压缩器解压。但是这样的话我认为接口本身就不应该提供源矩形大小的参数，所以最终可能是为了兼容不同平台上 D3D12 的接口才做了如此处理，最终导致了 DCC flag 有无造成了这样的 bug。
