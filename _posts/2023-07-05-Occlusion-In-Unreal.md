---
title: Occlusion Cull
author: Yohiro
date: 2023-07-05
categories: [UnrealEngine, Graphics, Rendering]
tags: [graphics, engine, unrealengine, occlusion query]
render_with_liquid: false
img_path: /assets/images/OcclusionQuery/
---

## 概述

最近需要利用 Occlusion Query 记录场景中可见物件的信息。为了验证方案的可行性，我决定先在开启 Occlusion Cull 的情况下，从引擎的 View.PrimitiveVisibilityMap 中回读 PrimitiveIndex 来确定可见物件。

然后发现 5.3 版本 SceneVisibility 这块变动挺大的，很多面熟的老函数不见了。查提交发现在引擎 5.3.2 版本中，可见性这块进行了重构，详情可见[这个提交](https://github.com/EpicGames/UnrealEngine/commit/9cd755694f97946ad0e84806250d9fdf428cefc7#diff-f521e57df7b2dd21cce113d087ba67ccadedb1b5c479916e2be97dfab6fd1caf)。该提交将 Occlusion Query 任务分配到了异步的 TaskGraph 中，还暂时移除了 PrecomputedVisibility（应该是 bug？）。

最后把 Occlusion Cull 这块重新过了一遍，这里留个档记录一下。

## Occlusion Cull

### 简介

Occlusion Query（遮挡查询） 或 Occlusion Cull（遮挡裁剪） 很多时候混为一谈，指的是对摄像机视锥内将被遮挡几何体剔除掉的技术，但是实际上是查询未必要剔除，剔除必然伴随着查询的。Query 是 GPU 提供的用于查询渲染资源状态的对象。通过 Query 我们可以查询到 Occlusion/ TimeStamp/ PSO状态 等。Occlusion Cull 是通过 Occlusion Query 是否有像素被绘制来确定是否被遮挡，从而决定几何体的可见性的。

一个简单的 Occlusion Cull 的大致流程如下：

1. 创建一个 Query 用作请求查询
2. 禁止颜色写入，所有通道的 ColorMask 设为 false
3. 禁止深度写入
4. 通知 GPU 开始 Query，此时会重置可见像素的数量
5. 在一个 Depth Only 的 Pass 中渲染场景中物体的包围盒做深度测试
6. 结束 Query，停止记录可见像素的数量
7. 开启颜色写入（如有必要，也可开启深度写入）
8. 获取 Query 的结果，即记录的可见像素的数量
9. 根据可见像素的数量判断该物体是否绘制

### 问题

上述的 Occlusion Cull 流程带来了两个问题，分别是 CPU 与 GPU 的同步问题以及遮挡细节带来的准确度问题。

#### 同步

CPU 和 GPU 的任务之间并不是同步进行的。比如，当 CPU 给 GPU 创建一个渲染指令时，CPU 不会在原地等待 GPU 的任务执行完成，而是继续去执行 CPU 上的其他任务，创建的渲染指令会被驱动程序发送给 GPU 的 CommandQueue，当这些命令执行完毕，便完成渲染。

但 Occlusion Query 的流程的问题在于，CPU 需要回读 GPU 端渲染包围盒的结果后才知道是否可见，从而进行裁剪。这意味者 CPU 必须在某一刻等待 GPU 完成查询，这个过程破坏了二者的并行性，从而大大降低了渲染效率。一个常用的解决方式是让 CPU 回读上一帧的 Query 结果，如果相机运动过快可能会出现错误的渲染，但由于 Occlusion Query 是基于包围盒进行裁剪，是一种保守的裁剪方式，因此影响不是非常明显。

![sync](await.jpg)
_CPU/GPU 以及 Occlusion Cull 的同步_

之前玩对马岛时，在室内场景快速旋转相机，有时会出现白色的谜之闪烁，应该就是和 Occlusion Query 的同步有关。

#### 遮挡

另一个带来的问题是使用包围盒来确认可见性带来的不精确的遮挡，考虑下图的这种情况：
![Occlusion Issue](fig01.jpg)

树的包围盒遮挡了车的包围盒，但是对于实际的像素而言，车并没有完全被树所遮挡。

一种解决方式是将一些几何体作为 Occluder 单独渲染，不经过遮挡剔除。渲染完成后，再针对较小的几何体做遮挡剔除。

### 效率

在虚幻的官方文档[^OffcialDocument]中，对引擎内各种裁剪方式进行了简单的介绍，在 Hardware Occlusion Queries 中提到：
> The cost of hardware occlusion scales with the number of queries performed on the GPU. Use Distance and Precomputed Visibility methods to reduce the number of queries performed each frame by the GPU.

也就是说，在使用 Occlusion Query 时应该注意一个平衡：

*通过 Occlusion Culling 裁剪获得的性能提升应该大于 Occlusion Test 产生的额外性能消耗，否则 Occlusion Query 的引入会对性能产生负面影响。*

## 虚幻中的 Occlusion Query

UE 在 Occlusion Query 之前，在 FrustumCull 中完成了 Distance Cull 然后在后续完成了 Precomputed Visibility 的检查，引擎采用这些操作以减少 Occlusion Test 过程中产生的 Draw call。此外，如前文所说，为解决 Occlusion Query 产生的 CPU/GPU 同步点的问题，虚幻采用的也是回读历史帧查询结果的方式，通过获取到的历史帧结果，以最大限度的减少 Occlusion Test 的消耗。

### 大致流程

该流程参考基于 Unreal Engine 5.3.2 版本。

#### 准备渲染

在当前版本，重构后的 LaunchVisibilityTasks 替代了原先的 ComputeViewVisibility。Visibility 的计算被分解为一个个 FVisibilityViewPacket 分配到 TaskGraph 中异步处理。

```cpp
void FVisibilityTaskData::LaunchVisibilityTasks()
{
    // ...

    // All task events are connected to prerequisites now and can be safely triggered.
    Tasks.BeginInitVisibility.Trigger();
    Tasks.LightVisibility.Trigger();
    Tasks.FrustumCull.Trigger();
    Tasks.OcclusionCull.Trigger();
    Tasks.ComputeRelevance.Trigger();
}
```

每个 FVisibilityViewPacket 会先做 Frustum Cull 筛掉不在视锥内和被 Distance cull 裁掉的几何体，然后再根据 Precomputed Visibility 筛掉一部分。

然后在等待上一轮 OcclusionQuery 的结果返回后，创建新一轮的 OcclusionCullTask:

```cpp
FVisibilityViewPacket::FVisibilityViewPacket(FVisibilityTaskData& InTaskData, FScene& InScene, FViewInfo& InView, int32 InViewIndex)
{
    // ...
    if (TaskConfig.Schedule == EVisibilityTaskSchedule::Parallel)
    {
        // Chain the frustum cull task to the relevance task since we only wait on relevance.
        Tasks.ComputeRelevance.AddPrerequisites(Tasks.FrustumCull);

        // Callback for when an occlusion command is queued from frustum culling.
        OcclusionCull.CommandPipe.SetCommandFunction([this](FPrimitiveRange PrimitiveRange)
        {
            UpdatePrimitiveFading(Scene, View, ViewState, PrimitiveRange);

            const int32 NumCulledPrimitives = PrecomputedOcclusionCull(*this, PrimitiveRange);

            if (OcclusionCull.ContextIfParallel)
            {
                OcclusionCull.ContextIfParallel->AddPrimitives(PrimitiveRange);
            }
        // ..
    }
}

void FGPUOcclusionParallel::AddPrimitives(FPrimitiveRange PrimitiveRange)
{
    WaitForLastOcclusionQuery();

    for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap, PrimitiveRange.StartIndex); BitIt.GetIndex() < PrimitiveRange.EndIndex; ++BitIt)
    {
        FGPUOcclusionParallelPacket* Packet = Packets.Last();

        if (Packet->AddPrimitive(BitIt.GetIndex()))
        {
            if (Packet->IsFull())
            {
                Packet->LaunchOcclusionCullTask();
                CreateOcclusionPacket();
            }
        }
        else
        {
            // The primitive will not be occluded, so accumulate a packet of primitives to send directly to the relevance pipe to reduce latency.
            NonOccludedPrimitives.Emplace(BitIt.GetIndex());

            if (NonOccludedPrimitives.Num() == MaxNonOccludedPrimitives)
            {
                ViewPacket.Relevance.CommandPipe.AddNumCommands(1);
                ViewPacket.Relevance.CommandPipe.EnqueueCommand(MoveTemp(NonOccludedPrimitives));
                NonOccludedPrimitives.Reserve(MaxNonOccludedPrimitives);
            }
        }
    }
}
```

在 LaunchOcclusionCullTask 后，FGPUOcclusionPacket::OcclusionCullPrimitive 中实现了具体的行为：

1. 首先判断是否有对应的 Occlusion History，若没有，直接创建一个

2. 如果有 Occlusion History 获取相应的 FRHIRenderQuery，根据 depth 的采样数量判断是否有像素绘制，从而判断是否被遮挡

3. 判断是否允许包围盒测试

    判断依据按优先级依次为：

    - 和摄像机原点的距离是否大于等于 `r.NeverOcclusionTestDistance` 设置的距离
    - 是否在近裁切面之后
    - 正交投影的情况下，是否位于相机投影体范围内（默认正交投影体是 XOY 平面 沿 Z 轴方向）
    - 其余情况下，外接球的半径是否小于 HALF_WORLD_MAX，关于该值的定义详见 EngineDefines.h

    如果包围盒测试未通过，则认定为未被遮挡，不会进入后续的 Occlusion Query 流程。

4. 通过包围盒测试后，首先更新用于记录上一次通过包围盒测试的帧数计数器。然后根据 Occlusion 的类型对 Bounds 进行注册，Occlusion 的类型以及优先级为

    - Occlusion Feedback
    - HZB Occlusion
    - 常规 Occlusion

5. 针对需要做常规 Occlusion 的几何体，根据 PrimitiveOcclusionHistory 里的遮挡情况进一步进行划分为：

    Grouped Occlusion
    : 历史帧结果是被遮挡的，会被视作不可见，将它们的 AABB 合为一批进行 Query，进行粗略的查询

    Individual Occlusion
    : 没有被遮挡的，需要使用单独的遮挡做查询

    在一个 Vertex Buffer 中 Grouped Occlusion 最大的 Batch 数不超过 16，Individual Occlusion 只有 1：

    ```cpp
    /** 
     * Initialization constructor. 
     * @param InView - copy to init with
     */
    FViewInfo::FViewInfo(const FSceneView* InView)
        :   FSceneView(*InView)
        ,   IndividualOcclusionQueries((FSceneViewState*)InView->State,1)
        ,   GroupedOcclusionQueries((FSceneViewState*)InView->State,FOcclusionQueryBatcher::OccludedPrimitiveQueryBatchSize)
        ,   CustomVisibilityQuery(nullptr)
    {
        Init();
    }
    ```

    ```cpp
    /** The maximum number of consecutive previously occluded primitives which will be combined into a single occlusion query. */
    enum { OccludedPrimitiveQueryBatchSize = 16 };
    ```

6. 划分完成后通过 BatchPrimitive 进行合批，同时将创建的 Query 注册到 PrimitiveOcclusionHistory中

7. 更新 PrimitiveOcclusionHistory

    刷新 PrimitiveOcclusionHistory 的`上次更新时间（LastConsideredTime）`，上次更新帧数以及上帧是否被遮挡等信息。其中`上次更新时间`是 FSceneViewState::TrimOcclusionHistory 清除旧 Queries 的重要依据。

    这一步主要是为了判断历史帧的结果是否有效，是否需要丢弃重新创建 Query。

8. 处理具有多个 Query 的几何体

    对于 HISM 这种有多个子 Bounds 的几何体，判断是否接受这一轮 Query 的结果。如果所有的子 Bounds 都被遮挡，直接标记它为不可见。

#### 渲染时

渲染时的入口可以在 DeferredShadingRenderer::RenderOcclusion。

1. 首先看是否 Depth Target 是否需要 down sample，然后对每个 View 去发起 Occlusion Query。

    调用 AllocateOcclusionTests 发起 Query 时，会先将失效的 Occlusion 结果通过 TrimOcclusionHistory 进行清除。

    ```cpp
    // Clear primitives which haven't been visible recently out of the occlusion history, and reset old pending occlusion queries.
    ViewState->TrimOcclusionHistory(ViewFamily.Time.GetRealTimeSeconds(), ViewFamily.Time.GetRealTimeSeconds() - GEngine->PrimitiveProbablyVisibleTime, ViewFamily.Time.GetRealTimeSeconds(), ViewState->OcclusionFrameCounter);
    ```

    清除的条件会参考引擎配置中的一个参数，叫做`潜在可见时间（PrimitiveProbablyVisibleTime）`，指的是明确在上一次可见后过多久可以认为该几何体依然可见，引擎默认是 8.0。

    如果在 `LastConsideredTime` 后过去了 `PrimitiveProbablyVisibleTime` 这么久，那么该几何体将从 Occlusion 的历史帧中被移除。

    需要注意的一点是，不光 BasePass，其他的渲染特性，如 Shadow，Light，Planar Reflection 等也要在这里创建 Query，也就是说会逐 View 的去创建 QueryArray。

2. 然后 VS 合批后在 BeginOcclusionTests 中执行绘制，在 FenceOcclusionTests 中执行同步。

    禁止写颜色和写深度：

    ```cpp
    FGraphicsPipelineStateInitializer GraphicsPSOInit;
    RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);
    GraphicsPSOInit.PrimitiveType = PT_TriangleList;
    GraphicsPSOInit.BlendState = TStaticBlendStateWriteMask<CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE>::GetRHI();
    // Depth tests, no depth writes, no color writes, opaque
    GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_DepthNearOrEqual>::GetRHI();
    GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = GetVertexDeclarationFVector3();
    ```

    整个 Query 过程只需要 AABB 与像素无关，可以不绑 PS 使用固定的 PSO：

    ```cpp
    // Lookup the vertex shader.
    TShaderMapRef<FOcclusionQueryVS> VertexShader(View.ShaderMap);
    GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();

    if (View.Family->EngineShowFlags.OcclusionMeshes)
    {
        TShaderMapRef<FOcclusionQueryPS> PixelShader(View.ShaderMap);
        GraphicsPSOInit.BoundShaderState.PixelShaderRHI = PixelShader.GetPixelShader();
        GraphicsPSOInit.BlendState = TStaticBlendState<CW_RGBA>::GetRHI();
    }

    SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit, 0);
    ```

几个关键的 Code path 是 RenderOcclusion -> AllocateOcclusionTests -> TrimOcclusionHistory -> BeginOcclusionTests -> FenceOcclusionTests。

### 其他

- 使用 down sample 的 depth target 去绘制 Occlusion

    使用 `r.DownsampledOcclusionQueries` 来开启/关闭使用半分辨率的 Depth Target 来渲染包围盒。

    ```cpp
    const auto RenderOcclusionLambda = [&]()
    {
        RDG_GPU_STAT_SCOPE(GraphBuilder, RenderOcclusion);

        const int32 AsyncComputeMode = CVarSceneDepthHZBAsyncCompute.GetValueOnRenderThread();
        bool bAsyncCompute = AsyncComputeMode != 0;

        FBuildHZBAsyncComputeParams AsyncComputeParams = {};
        if (AsyncComputeMode == 2)
        {
            AsyncComputeParams.Prerequisite = ComputeLightGridOutput.CompactLinksPass;
        }

        RenderOcclusion(GraphBuilder, SceneTextures, bIsOcclusionTesting,
            bAsyncCompute ? &AsyncComputeParams : nullptr);

        CompositionLighting.ProcessAfterOcclusion(GraphBuilder);
    };
    ```

- 缓存的 Occlusion 历史帧不超过 4 帧

    每个 FPrimitiveOcclusionHistory 都持有一个相对应的 FPrimitiveComponentId，并记录了其过去几帧（不大于 4）的 Query 结果。

    ```cpp
    //
    // ScenePrivate.h
    //
    /** The occlusion query which contains the primitive's pending occlusion results. */
    FRHIRenderQuery* PendingOcclusionQuery[FOcclusionQueryHelpers::MaxBufferedOcclusionFrames];
    uint32 PendingOcclusionQueryFrames[FOcclusionQueryHelpers::MaxBufferedOcclusionFrames]; 

    //
    // ScenePrivateBase.h
    //
    enum
    {
        MaxBufferedOcclusionFrames = 4
    };

    // get the system-wide number of frames of buffered occlusion queries.
    static int32 GetNumBufferedFrames(ERHIFeatureLevel::Type FeatureLevel);
    ```

- 历史帧结果，固定每 6 帧清理一次

    ```cpp
    void FSceneViewState::TrimOcclusionHistory(float CurrentTime, float MinHistoryTime, float MinQueryTime, int32 FrameNumber)
    {
        // Only trim every few frames, since stale entries won't cause problems
        if (FrameNumber % 6 == 0)
        {
            int32 NumBufferedFrames = FOcclusionQueryHelpers::GetNumBufferedFrames(GetFeatureLevel());

            for(TSet<FPrimitiveOcclusionHistory,FPrimitiveOcclusionHistoryKeyFuncs>::TIterator PrimitiveIt(Occlusion.PrimitiveOcclusionHistorySet);
                PrimitiveIt;
                ++PrimitiveIt
                )
            {
                // If the primitive hasn't been considered for visibility recently, remove its history from the set.
                if (PrimitiveIt->LastConsideredTime < MinHistoryTime || PrimitiveIt->LastConsideredTime > CurrentTime)
                {
                    PrimitiveIt.RemoveCurrent();
                }
            }
        }
    }
    ```

- SceneProxy 的 AcceptOcclusionResults 接口
    这个虚函数默认没有实现，目前引擎中只有 HISM 和 VirtualHeightFieldMesh 实现了这个接口。

    AcceptOcclusionResults 能够提供当帧的 Occlusion 结果。对于 HISM 来说，它内部维护了一个 ViewId 和 FFoliageOcclusionResults 的 Map，如果 SubOcclusionQueries 与 HISM 的 FFoliageOcclusionResults 不匹配的时候，根据 AcceptOcclusionResults 的结果重建该 Map。可能因为存在这种 Clustering 概念的几何体集合，所以引擎的代码里才引入了 SubQueries 这个概念。

- Nanite mesh 走 GPU 剔除的 pass，如果使用 FreezeRendering 调试会发现 nanite mesh 不会被裁剪。此外有些渲染特性的开启也会导致相应的 primitive 不会被遮挡裁剪剔除掉，具体的可以查看相应的 SceneProxy 的 CanBeOccluded 函数。

- Mobility 为静态（static 以及 Stationary）和纯动态的组件，Occlusion Query 的处理不同，笼统地说，动态组件的查询会更精确，因为动态的 primitive 不会使用 grouped，将作为独立的 primitive 去查询。

## 参考

- [chapter-29-efficient-occlusion-culling](https://developer.nvidia.com/gpugems/gpugems/part-v-performance-and-practicalities/chapter-29-efficient-occlusion-culling)
- [Castle Game Engine - Occlusion Culling](https://castle-engine.io/occlusion_culling?page=/occlusion_query)
- [Query Object](https://www.khronos.org/opengl/wiki/Query_Object#Occlusion_queries)
- [Chapter 6. Hardware Occlusion Queries Made Useful](https://developer.nvidia.com/gpugems/gpugems2/part-i-geometric-complexity/chapter-6-hardware-occlusion-queries-made-useful)
- [剔除：从软件到硬件](https://zhuanlan.zhihu.com/p/66407205)
- [18.3. Occlusion Queries](https://registry.khronos.org/vulkan/specs/1.3-extensions/html/vkspec.html#queries-occlusion)

[^OffcialDocument]:  [Visibility and Occlusion Culling](https://docs.unrealengine.com/5.3/en-US/visibility-and-occlusion-culling-in-unreal-engine/)