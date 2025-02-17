---
title: World Building Guide
author: Yohiro
date: 2023-06-09
categories: [UnrealEngine]
tags: [unrealengine, world partition, streaming]
render_with_liquid: false
media_subpath: /assets/images/{}/
---

## [**World Partition**](https://docs.unrealengine.com/5.1/en-US/world-partition-in-unreal-engine/)

### 实践

#### Grid setup

- 以单独的 Grid 开始构建，最终的成品中包含的 Grid 越少越好。
- 应基于 Gameplay、物理行为、Actor 数量等来规划 Grid 的具体的尺寸和加载范围，并在规模变化时进行 profiling，再次调整。
- 考虑 HLOD

#### 使用 Level Instance 和 Packed Level Actor

#### 使用 StreamingSourceComponent

在 5.1 以后的版本中，PlayerController 已经是一个 StreamingSourceComponent。

``` cpp
class ENGINE_API APlayerController : public AController, 
                                     public IWorldPartitionStreamingSourceProvider
```

这个组件可以附加在任何 Actor 上，可以用于一些无缝 CutScene 的预加载。

### 问题

#### 创建许多新 Grid 以及运行时的 Data Layer

- 每个 Grid 的每一个小格子都至少包含一个 Actor，产生一个 StreamingLevel。此外，每个 Data Layer 也会创建一个 StreamingLevel。任何行为产生的 StreamingLevel 都会对性能有所影响，例如：
        - 根据资产类型创建多个 Grid 或 Data Layer，像是在一个森林的场景中创建树木的 Grid、FX 的 Grid、灯光的 Grid、大石头小石头的。
        - 仅使用 Data Layer 用作编辑目的而不在运行时改变其状态。

- 需要平衡 Grid/Data Layer 的规模和 Streaming 的效率
        - 通常为一些特殊内容添加 Grid/Data Layer 的行为，不会造成太大的性能影响，可以放心使用。

#### 场景/Data Layer 的组织

#### Actor 的包围盒

### 限制

#### Grid 隶属于 Persistent 关卡

目前仅支持一个 Persistent 关卡中包含多个 Grid。

#### 2D Streaming

当前版本的 Streaming 是 2D 视角的，不支持垂直空间的 Streaming。根据 World Partition 的 [Roadmap](https://udn.unrealengine.com/s/article/World-Building-Features-Roadmap)，将在 5.4plus 版本中添加 3D 区块的支持。

#### 重新配置区块 vs 分割 Actor

分割 Actor，例如 Landscape、Foliage 时，不能超过区块大小的限制，否则导致该 Actor（Landscape/Foliage）在每次 Streaming grid 重新配置后无法更新。当前版本的引擎内提供了命令行重新创建这类 Actor，需要时可以调用。

#### 编辑器内需要手动加载

当前版本中并不支持根据相机的位置、朝向自动加载区块，需要设计者在 World Partition 编辑器内选择加载。相较于 Nanite 和 Texture 的 Streaming，World Partition 在编辑器内的加载不受 Pool/Buffer 数量的限制，因此会导致内存耗尽从而崩溃。

#### 运行时限制

- 运行时无法创建/修改 grid，生成 streaming 的行为法中在 PIE 和 Cook 时。
- 不支持运行时关卡的创建/注入行为
        - 修改 Game Feature 插件可以往指定的关卡添加 Actor
        - 修改 Level Instance 的代码也是可行的
- Spawn 的 Actor 是位于 Persistent 关卡中的，子 Actor 的关卡位置取决于其父节点的加载关卡
- Persistent 关卡中的 Actor 引用其他 Spatially Loaded 的 Actor，引擎禁止该行为

#### GC

为了分散 GC 的压力，已卸载的内容会在项目配置中定义的固定时间片后进行 GC，或是当指定数量的 cell 被卸载后进行 GC。

#### One File Per Actor（OFPA）

WP 与 OFPA 密不可分，因此需要考虑产生的额外的 actor 文件数量所带来的问题，还有 source control 所面临的问题。

#### Hot reload

当 Sync/Revert 一个资产时，引用到该资产的 Actor，如果位于 WP 中，hot reload 不会对这个 Actor 起效。WP 关卡必须关闭然后重新加载 Actor。

#### 服务器端的 Streaming

当前版本不支持，根据 [**Roadmap**](https://udn.unrealengine.com/s/article/World-Building-Features-Roadmap) 显示，将在 5.3 版本中添加该功能（Experimental），5.4plus 版本中可供产品使用。

#### 光照

仅支持动态光照，不允许烘培静态光。

### 用例

#### 堡垒之夜 第四章

|地图大小|`2km x 2km`|
|Actor 数量|`约 100k`|
|Cell 大小|`128m`|
|加载范围|`256m（加载距离随平台和性能等级改变）`|

- 2 HLOD setup
        - 建筑物的 HLOD（两层）
            - HLOD 0
                为支持破坏功能，HLOD0 的 mesh 被合并。256m cell，加载范围 512m，spatially loaded
            - HLOD 1
                简化的 mesh，512m cell，2048m load range，spatially loaded
        - 树木的 HLOD（一层）
            - 使用 imposter 的 Instanced layer , always loaded

- Level Instances:
        - 所有的 POI
- Data layer：
        - 4 个 初生岛屿和团队战大厅
        - 1 个 用于赛季变换/特殊事件
- 服务器端：
        - 全部加载
        - Server streaming 在 PIE 下使用，用于提升开发者的迭代效率
- Landscape:
        - Always loaded
- Packaed Level Actors:
        - 无

#### 黑客帝国 demo

|地图大小|`4km x 4km`|
|Actor 数量|`约 107k`|
|Cell 大小|`128m`|
|加载范围|`128m`|

- HLOD setup
        - HLOD 0
            启用 Nanite 的 Instance layer，256m cell，加载范围 768m，spatially loaded
        - HLOD 1
            简化的 mesh，256m cell，always loaded
- 使用 houdini 过程化构建，同时使用 Rule Processor 插件
        - 所有的 POI
- Packaed Level Actors:
        - PLA 和使用了 Instanced static mesh 过程化生成的 Actor
        - 该做法应用于所有的建筑物、街道、小物件
- Data layer（运行时 35/ 编辑器 21）：
        - 用于 cut scenee 和 gameplay
        - 用于屋顶和路面下的资产
- Landscape:
        - 无

#### 古代山谷 demo

|地图大小|`2km x 2km`|
|Actor 数量|`约 14k`|
|Cell 大小|`64m`|
|加载范围|`64m`|

- HLOD setup
        - HLOD
            Instance layer，always loaded
- Packaed Level Actors:
        - 用于组合岩石
- Data layer（运行时 2/ 编辑器 1）
- Landscape:
        - 无

### 调试

- 常用 CVars

    更多的可以执行 DumpConsoleCommands，将所有命令 dump 出来，wp 相关的 debug 指令太多，下面仅列出一些常用的。

| wp.Runtime.ToggleDrawRuntimeHash2D or 3D                                              | streaming grid 可视化 | 
| wp.Runtime.OverrideRuntimeSpatialHashLoadingRange -grid=[index] -range=[DesiredValue] | 覆写指定 grid 的加载距离 |
| wp.runtime.hlod                                                                       | 显示 HLOD |
| wp.Runtime.RuntimeSpatialHashUseAlignedGridLevels                                     |       |
| wp.Runtime.RuntimeSpatialHashSnapNonAlignedGridLevelsToLowerLevels                    |       |
| wp.Runtime.RuntimeSpatialHashPlaceSmallActorsUsingLocation                            |       |

## [**One File Per Actor**](https://docs.unrealengine.com/5.1/en-US/one-file-per-actor-in-unreal-engine/)

One File Per Actor(OFPA)

### 实践

#### Epic 强建议 WP 的使用者使用引擎内部的版本管理

原因如下：

1. WP 所引用的外部 Actor 文件以 GUID 命名，因此在外部文件夹中或是版本控制软件中难以识别。而在引擎的版本控制视图下，可以看到 Actor 的名称、类型以及路径，允许使用者对这些文件进行过滤、排序。
2. Uncontrolled CL 可以追踪所有本地非只读的文件，这一功能在外部的版本控制（如 Perforce）中不可用。
3. 在提交时存在数据验证阶段，而且这里的验证可以根据项目需要进行扩展。
4. 在引擎内通过 CL 导航至 Actor 或其他内容，比起切换到外部的版本控制软件，体验上更加丝滑。

#### 使用 Uncontrolled CL（ChangeList）

- 使用 Uncontrolled CL 可以在不打开版本控制软件的情况下看到你编辑的、设为可写的或是添加的资产。
- 能使 workspace 的使用情况更加清晰，避免出现不受版本控制的文件的影响。
- 当修改内容准备完毕，将 Uncontrolled CL 移到版本控制的 CL。
- 程序同学应该使用 Uncontrolled CL 进行调试、测试以避免版本控制对 binary 的锁定（比如 check out）。

#### 创建自定义的提交验证

提交验证可以通过代码进行扩展，关键的验证如 CL 中的文件缺失、资产引用问题，这些已集成到引擎中。

### 问题

#### 需要使用者去管理个人文件

大多数使用 OFPA 的人并不想管理一个又大又长的 Changelist，并且担忧与同事并行工作会带来问题。

#### 使用 P4V 管理外部 Actor

如上文实践中所述，使用 WP 的过程中应该尽量避免使用 P4 提交。

#### 分支/合并/创建副本/文件数量

- OFPA 会给服务器端的加载、同步以及创建分支添加很大的压力，需要 DM 多加注意。
- 对大型工作室，应考虑代理、edge server 等方式分散负载。

### 限制

#### GUID 只能在 UE 里转为可读的名称

#### 启用 OFPA vs 不启用 OFPA

对于启用和不启用 OFPA 的 Level Instances，他们的 Streaming 行为有所不同：

启用 OFPA：
: 在 PIE/Cook 生成流送数据时，Level Instance 中的内容会被分散开放到 Persistent 关卡，并被指定到相应的 Grid 中。而 Level Instance 本身在运行时不存在。

不启用 OFPA
: 在 PIE/Cook 生成流送数据时，Level Instance 会被保留，而且会被当作一个 Actor 在包含该 Level Instance 的 grid cell level 被加载的同时 Stream in。

#### 访问未加载 Actor

当一些工具需要获取 Actor 的 GUID 时，如果所在 World 没有加载，GUID 信息/Actor 的描述信息是取不到的（以前不也是这样吗？）

#### 锁定

这里的锁定，指的是版本控制中的锁定，是指阻止用户编辑一个没有权限的文件（一般是被其他人 CheckOut 的文件）的行为。当前没有实现 Actor 锁定。

#### 观察文件状态

相较于 5.0 版本，5.1 在获取文件状态方面做出了一些提升。用户确认是否可以对资产做出修改的一种方式是：在场景大纲下启用版本控制列，观察是否存在未保存的文件与版本控制内的文件有冲突。

## [**Level Instances 和 Packed Level Actors(PLA)**](https://docs.unrealengine.com/5.1/en-US/level-instancing-in-unreal-engine/)

Level Instance 和 Packed Level Actor 是 5.x 版本中实现 Level Instance 的两种方式，前者是 Level 级别的粒度，后者是 Actor 级别的粒度。

### Level Instance

创建子关卡的 Actor 的集合，与 4.x 版本中基于关卡的 workflow 类似。当使用 World Partition 时，Level instance 应该启用 OFPA，因为启用 OFPA 后，引擎会在生成 Streaming 数据时将 Level instance 中的内容分离到 Persistent 关卡中的 Streaming grid。如果 Level instance 不启用 OFPA，那么它会被当作一个独立的 Streaming level，在规模较大较复杂的场景中会导致性能和 Streaming 的问题。

#### 特性

- 具有可嵌套的层级结构
- 同一 World 中现支持多个 Level instance
- 对非 OFPA 关卡支持 Level streaming
- 对关卡整体设置的编辑
- Level instance 内的所有 Actor 都支持 Data layer

#### 适用场景

POI、房屋、内部陈设、建筑物的地板、村庄、Gameplay 相关配置等。

### Packed Level Actor

一种由`Static mesh`（包括 ISM/HISM）合并而来的`Blueprint actor`，Static mesh 会被替换为链接到`Packed level actor`的`Packed level blueprint`。简单来说 PLA 是由许多 ISM/HISM 组件构成的 Actor。PLA 不可覆盖不可脚本话，它会在每次更新时重新创建，所以应应用于静态物件。

#### 特性

#### 适用场景

静态建筑物、模型复用率高的拼合密度较高的大型物件等。

### 问题

#### 创建具有 level instance 的 world scale layer

由于编辑器中加载 level instance 的过程是阻塞、非异步的，使用非常大尺寸的 level instance 的去嵌套其他所有的 POI 往往会造成加载问题。必要时在所需的位置放置较小的 level instance，使用 Editor only 的 Data layer 是个比较好的选择。

#### 使用非 OFPA 的 level instance

World 中每个没有启用 OFPA 的 level instance 都会创建一个独立的 streaming level，这会导致 streaming 问题和性能问题。

#### PLA 过大

当 PLA 的大小超过了 streaming cell 的大小，会引发一些性能问题、内存问题，正常情况下 PLA 的大小不应超过 streaming cell 的大小

### 限制

#### 无法覆盖

- 对 level instance，所有编辑的结果会直接应用到关卡的源文件上。
- 对 Packed level actor，每次更新 PLA 时，PLA 都会重新构建，所以当覆盖一个属性时会触发该机制从而无法覆盖。

#### 不支持关卡蓝图

由于 level instance 在 streaming 期间会被分离为 persistent 关卡中的各个资产，所以无法执行单一关卡的脚本。

#### level instance 中 actor 的引用

无法从高级别的 actor 引用 level instance 内的子 actor

#### 编辑器内无异步加载/部分加载

- 在编辑器中加载 level instance 会加载其所有的内容，整个过程是非异步的。
- PLA 会被当作单独的 Actor 加载。

#### 非编辑模式下无法获取子 Actor

- 目前的版本中，level instance 的子 Actor 无法被选中、编辑、拷贝、查看属性等
- 这会给一些操作带来不便，比如在主 world 编辑地形时，尝试隐藏部分 level instance 的内容，或是复制其他 level instance 粘贴到当前编辑的关卡中。

### 用例

#### 堡垒之夜 第四章

- 场景中所有的主要位置，如建筑物、房屋、POI 这种粒度的关卡，都由 level instance 构建。
- 大多数情况下 level instance 被当作独立的关卡。
- 没有使用 PLA，因为堡垒之夜的 gameplay 决定了，建筑物可以破坏、交互，因此没有使用 PLA。

#### 黑客帝国 demo

- PLA 用于除程序化生成的建筑物外其他的主要建筑物。
- PLA 还用于创建预制的屋顶
- 所有过程化生成的建筑物、道路、屋顶、部分物件都由 ISM 构成（等同于 PLA·
- 只有最基础的关卡保有逐模块的碰撞。
- 产品的最后阶段，PCG 流程停止了，PLA 最终也被转换为了 ISM actor。这么做都是为了能够手动修改物件，但该过程是不可逆的。

#### 古代山谷

- PLA 扩展为可支持碰撞合并，以优化 streaming level 时在物理模块和 AddToWorld 时的消耗。

## [**Data Layers**](https://docs.unrealengine.com/5.1/en-US/world-partition---data-layers-in-unreal-engine/)

Data layer 允许在运行时/编辑时限定数据加载的条件。Actor 和 World Partition 决定了 streaming 的逻辑，Data Layer 则像是一个过滤器，用于决定哪些关卡需要加载。

### 运行时 Data layer

- 用于处理不同情景
- 管理任务、游戏进度、事件等各种特定的数据
- HLOD 支持，创建的 HLOD 的状态会和 Data layer 的状态一同变化同时也是编辑器的 Data layer
- 运行时具有三种状态：
        - Unloaded（unloaded and not visible）
        - Loaded（loaded and not visible）
        - Activated（loaded and visible）

### 编辑器 Data layer

- 用于在编辑时组织内容
- 为了更方便的编辑，数据相对独立
- 预览运行时 Data layer 的内容
- 存在仅编辑器可见的 Data layer，在 Cook 版本和 PIE 中不可见
- 编辑器 Data layer 的状态：
        - IsInitiallyVisible（加载 world 时，是否默认可见）
        - IsInitiallyLoaded（加载 world 时，是否默认加载）
        - Loaded
        - Visible

### 实践

#### 使用仅编辑器可见的 Data layer 来分离数据

使用编辑器 Data layer 可以将指定的数据如 gameplay sequence 和 cinematic 数据和其他的数据分离开。

#### 预加载

#### Data layer 的负责人

对于项目的 Data layer，最好有技术人员制定它的结构，有条件的话，可以预先定义 data layer 的资产以匹配项目的结构和目标，比如任务/事件/游戏进程/工作类型等。

#### 优化

通过 Data layer，可以在特定的 Sequence 或 gameplay 中移除不需要的关卡，也可以针对不同的平台进行优化，从而节省内存，提升性能。

### 问题

#### Data layer 的规模

- 运行时 Data layer 的创建应该被重点照顾，因为如 WP 一节所述，每一个 cell 中的 actor 对应的运行时 Data layer 都会创建一个新的 Streaming level。过多使用运行时 data layer 会严重影响 streaming 的效率。
- 不过使用运行时 Data layer 来处理特定的内容，比如每个任务或是某个主题的场景，是比较安全的行为，对全局的 streaming 效率没有太大影响。

#### Actor 上多个 Data layer 的组合

#### Data layer 和 Streaming 的混淆

#### 使用 Data layer 加载太多东西

### 限制

####

####

###

### 用例

#### 堡垒之夜 第四章

- 4 个运行时 Data layer，用于大厅/初始小岛
- 1 个运行时 Data layer，用于赛季变动以及特殊事件

#### 黑客帝国 demo

- 35 个运行时 Data layer
- 32 个编辑器 Data layer
        - Sequence 以及 Gameplay 内容的 streaming 和卸载
        - Sequence 特定的优化
        - 多个仅编辑器的 Data layer 用于 PCG 生成

#### 古代山谷 demo

2 个运行时 Data layer，1 个 Editor only

## [**HLOD**](https://docs.unrealengine.com/5.1/en-US/world-partition---hierarchical-level-of-detail-in-unreal-engine/)

WorldPartition HLOD 与原先 UE4.x 中的 HLOD 不同在于，WorldPartition HLOD 不是和单独的关卡相关联的，而是由 Grid 生成的。初此之外与原先的 HLOD 概念基本类似，都是 Actor 聚合所生成的低模代理。

### 实践

#### 预先定义 HLOD 的类型和层级

在概念阶段定义拥有最好的视觉效果的 HLOD 的 streaming、层级和类型，可以从运行平台、运行平台的内存和性能、Instancing、Nanite、场景密度、场景尺寸、WP 的 Grid 大小、资产类型等方面出发考虑 HLOD 的优化。

#### 在合适的距离评估 HLOD

如果使用 Merged 或 Simplified 类型，近距离的 HLOD 会显得非常潦草，因为 HLOD 被设计为用作远距离时的低模代理。启用 Nanite 的 Instance 层会获得不错的效果。
