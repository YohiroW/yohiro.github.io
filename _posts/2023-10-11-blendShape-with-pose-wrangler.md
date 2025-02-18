---
title: 使用 Pose Wrangler 创建 Blendshape
author: Yohiro
date: 2023-10-11
categories: [Animation, UnrealEngine]
tags: [animation, blendshape, maya, deform]
render_with_liquid: false
img_path: /assets/images/blendshape/
---


> 新版 Pose Wrangler 插件已经发布在了 Epic 商城中，新版插件添加了 Driver Bone 的导出模块，及其导入引擎的插件。可以[**点击这里**](https://www.unrealengine.com/marketplace/en-US/product/pose-driver-connect)获取新版插件。新版插件中对 BlendShape 的支持依然是 Experimental，具体的教程视频以及讨论串可以访问[**这个帖子**](https://forums.unrealengine.com/t/tutorial-authoring-secondary-animation-in-maya-and-unreal-with-pose-driver-connect/1496699)
{: .prompt-warning }

## 背景

Pose Wrangler 是 EpicGames 提供的Maya开源工具，原始的仓库在[**这里**](https://github.com/chrisevans3d/poseWrangler)。也可以[获取个人 fork 的版本](https://github.com/YohiroW/poseWrangler)，添加了对导出 fbx 以及 json 的支持。

使用 PoseWrangler 创建 Blendshape 有在 DCC 的准备步骤和导入引擎后的步骤，下面将从 Maya 和 UnrealEngine 两端进行说明。

## Maya

### 界面

根据文档，python 脚本编辑器内输入以下代码以打开插件界面，Driver

``` python
from epic_pose_wrangler import main
pose_wrangler = main.PoseWrangler()
```

可以保存到工具架，方便下次使用。

### 创建解算器

首先需要在 PoseWrangler 中根据骨骼创建 RBF 解算器，RBF 是指 *Radial Based Function*，即*径向基函数*，在建模领域主要应用于肌肉、表情等形变方向。

1. 点击 <kbd>Create Solver</kbd> 以创建解算器，初次创建的解算器会自动进入编辑状态
2. 如果没有进入编辑状态，点击 <kbd>Edit Solver</kbd> 以进入编辑状态（编辑完成需要手动点击 Finish Editing）

**Tips**：可以通过导入预设的 json 文件，以减少工作量。

### 创建 Pose

在创建完骨骼对应的解算器后，进入针对当前解算器的编辑状态。

1. 在大纲视图中选中驱动骨骼后，点击 <kbd>Add Driver Transform</kbd>
2. 点击 <kbd>AddPose</kbd> 以添加 Pose
3. 在 Maya 内的编辑视口内控制骨骼，调整为需要编辑的 Pose
4. 调整完成后，点击 <kbd>Update Pose</kbd> 更新 Pose
5. 修改 Pose 对应的名称

**Note**: 在 Pose 列表为空时，添加驱动骨骼时将会创建名为 default 的默认 Pose，如果需要使用插件编辑 blendshape，那么该 Pose 需予以保留。v1.0 版本中的默认 Pose 的命名 base_pose 在 v2.0 中将不支持。

### 创建 Blendshape

1. 选中当前的解算器和需要修型的 Pose
2. 在大纲视图中选中需要添加 Blendshape 的 Mesh 后，点击 <kbd>Create Blendshape</kbd>
3. 创建了 Blendshape 后会自动创建原 Mesh 的副本， 并且选中该副本进入编辑 Blendshape 的状态
4. 编辑完该 Pose 后，点击 <kbd>Finish Editing</kbd> 完成对 Blendshape 的编辑

### 导出 fbx

1. 点击 <kbd>Bake Poses To Timeline</kbd> 将当前解算器所对应的 Pose 作为帧动画烘焙到时间轴上。该动画会在导入引擎后产生 Anim Sequence 文件，我们需要使用 Anim Sequence 文件来创建引擎内可使用的 Pose 资产。
2. 选中骨骼和几何体导出为 fbx，确保 Animation 和 Blendshape 能够被导出。

## UnrealEngine

### 导入 fbx

1. 在 fbx 的导入选项中，确保勾选 `Import morph targets`
2. 如果需要将曲线导入骨骼中，要在 fbx 的导入选项中，确保勾选 `Add Curve Metadata to Skeleton`

如果正确的话，此时可以在 Skeletal Mesh 的 `Morph Target Preview` 标签窗口下看到所有在 Maya 中创建的 Blendshape。

### 准备 Pose 资产

需要借助动画蓝图中的 `Pose Driver` 节点驱动 Blendshape，而使用 PoseDriver 节点需要在引擎内创建 Pose 资产。

1. 首先需要确保，导入到引擎的动画序列中拥有动画曲线的信息
2. 在 ContentBrowser 里找到导入引擎中的动画序列 ，右键可以创建 Pose 类型的资产。以这种方式创建的 Pose 资产会将动画序列中的每一帧提取为一个 Pose ，其顺序与在 Maya 中 Pose Wrangler 的 Pose 列表一致
3. 按顺序输入 Pose 的名称即可

### 动画蓝图

测试用动画蓝图只包含两个节点

- InputPose 用于指定当前的 Pose
- Pose Driver 将会根据 Pose 资产中定义的一系列 Pose 去驱动指定的骨骼

测试用的 Pose Driver 节点的参数较为简单:

1. 指定 Pose 资产
2. 指定源骨骼为 Maya 中编辑的骨骼
3. RBF 参数中的解算半径设为自动
4. Pose Target 这里直接从 Pose 资产中拷贝

### 预览

将该动画蓝图作为 Skeletal Mesh 的后处理动画蓝图即可预览

效果如下：

| 无 Blendshape | 有 Blendshape |
|:--------------|:--------------|
| ![without blendshape](without_bs.png) | ![with blendshape](with_bs.png) |

## TroubleShooting

- 确保当前所使用的 PoseWrangler 的版本是 2.x 版本，非 2.x 版本没有创建 blendshape 的选项
- 需确保 Maya 中的各节点没有使用命名空间
- 需确保骨骼和 Mesh 的原点都位于（0，0，0）
- 创建的 Pose 时, 默认的 TPose 需更名为 `default`
