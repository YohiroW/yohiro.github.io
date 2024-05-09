---
title: DBSCAN 算法在引擎中的应用
author: Yohiro
date: 2024-02-15
categories: [Unreal]
tags: [algorithm, machine learning, unrealengine]
math: true
render_with_liquid: false
img_path: /assets/images/DBSCAN/
---
## 介绍

在[之前的文章](/posts/Clustering-DBSCAN)中介绍了 DBSCAN 的原理，这里来记录一下在引擎中的应用。

### 实现参考

这里的参考选择了 [SimpleDBSCAN](https://github.com/CallmeNezha/SimpleDBSCAN)，一个轻量的 header-only 的 C++ 实现。SimpleDBSCAN 的实现中使用了 [kd 树](https://oi-wiki.org/ds/kdt/)来做复杂的样本划分，以加速大样本的查询。

## 参数选取

1. 选取 k 值，建议取 k 为2*维度-1。（其中维度为特征数）
2. 计算并绘制k-distance图。（计算出每个点到距其第k近的点的距离，然后将这些距离从大到小排序后进行绘图。）
3. 找到拐点位置的距离，即为Eps的值。
4. 而 Minpts 的值为 k+1

![k-dist](k-distance.png)

## 应用

最初决定引入一个聚类的方式是为了尝试在 UE 中将放置某种包围盒的步骤自动化，从而提高迭代效率。

我在场景中沿着 Nav Mesh 生成了采样点，采样点的分布如下：

![Samples](Samples.png)
_可以视作采样点的集合 D_

使用 DBSCAN 进行聚类的结果如下，其中 MinPts = 10, Epsilon = 400:

| 聚类结果 | 噪声 |
| ![MinPts = 10, Epsilon = 400](MinPts=10_Eps=400.png) | ![Noises](MinPts=10_Eps=400_Noises.png) |

就该结果而言，存在下面几个问题：

1. 聚类可以进一步细分，比如最上面连成一大块的紫色可以进一步划分
2. 对于比较狭窄的地方，由于采样点不足，无法形成有效聚类，从而产生噪声
3. 对于比较宽阔的地方，由于采样点之间的距离比较大，无法形成有效聚类，从而产生噪声

在引擎中导出 k-distance 数据后，绘制了图像如下：

![k-distance](Kdist-Figure-5.png)

由于样本点数目较多，因此很难直观地看出拐点在哪里，缩小一下范围，合适的值的大概位于这个区间中：

![](slopest_range.png)

| 聚类结果 | 噪声 |
| ![MinPts = 6, Epsilon = 300](MinPts=6_Eps=300.png) | ![Noises](MinPts=6_Eps=300_Noises.png) |









## 参考

- [DBSCAN](https://zh.wikipedia.org/wiki/DBSCAN)
- [scikit-leran clustering](https://scikit-learn.org/stable/modules/clustering.html#dbscan)
- [基于密度的聚类算法（1）——DBSCAN详解](https://zhuanlan.zhihu.com/p/643338798)
- [常用聚类算法](https://zhuanlan.zhihu.com/p/104355127)
- [SimpleDBSCAN](https://github.com/CallmeNezha/SimpleDBSCAN)
- [DBSCAN Clustering Easily Explained with Implementation](https://www.youtube.com/watch?v=C3r7tGRe2eI)
- [DBSCAN的参数选择及其应用于离群值检测](https://blog.csdn.net/Cyrus_May/article/details/113504879)
