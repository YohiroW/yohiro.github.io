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

在[上一篇文章](/posts/Clustering-DBSCAN)中介绍了 DBSCAN 的原理，这里来记录一下在引擎中的应用。

我决定引入一个聚类的方式是为了尝试在 UE 中将放置某种包围盒的步骤自动化，从而提高迭代效率。

我在场景中沿着 Nav Mesh 生成了采样点，采样点的数量为 4643 个，分布如下：

![Samples](Samples.png)
_可以视作采样点的集合 D_

在参照下图对其他聚类算法进行了一些对比后，选择了使用 DBSCAN。

![cluster_comparison](https://scikit-learn.org/stable/_images/sphx_glr_plot_cluster_comparison_001.png)
_各种聚类算法的效果对比_

这个选择出于以下几个原因：

1. 采样点是沿着 Nav Mesh 生成的，密度比较平均
2. DBSCAN 的算法比较简单，易于实现
3. 游戏中的路径形成的样本集有可能是非凸的
4. 噪声多可以接受，身是可以通过去除噪声点而达到离线构建效率优化的目的

综上，我选择了 DBSCAN。

## 参数选择

之前的文章中提到过参数选择的问题，[原论文](https://cdn.aaai.org/KDD/1996/KDD96-037.pdf)中介绍了一种叫做 **k-dist** 的选择 Epsilon 和 MinPts 的方法。

### k-dist

**k-dist** 指的是样本中的点 P 到距它第 k 近的点的距离。

设 d 是点 p 与其 k 个最近邻居的距离。
则几乎所有点 p 的 d 邻域都恰好包含 k+l 个点。
的 d 邻域包含多于 k+l 个点，前提是有几个点
与 p 的距离 d 完全相同，但这种可能性很小。

对于给定的 k，我们定义一个从数据库
D 到实数的函数，将每个点与其第
的距离。将数据库中的点按
按照 k-dist 值从大到小的顺序排列时，该函数的图
该函数的图给出了数据库密度分布的一些提示。我们称这个图为排序后的 k-dist
图。如果我们选择一个任意点 p，将参数
Eps 为 k-dist(p)，并将参数 MinPts 设为 k，则所有点的
将成为核心点。如果
如果我们能在 D 的 “最薄 ”聚类中找到一个 k-dist 值最大的阈值点，我们就能得到所需的参数值。
参数值。阈值点就是
第一个 “谷 ”中的第一个点（见图 4）。所有
所有 k-dist 值较高的点（阈值左侧）都被认为是噪音，所有其他点都被认为是噪音。
所有其他点（阈值右侧）都会被归入某个集群。

DBSCAN 需要两个参数，即 Eps 和 MinPts。然而，我们的实验表明，k > 4 时的 k-dist 图与 4-dist 图没有明显区别，而且它们需要的计算量要大得多。
的 k-dist 图与 4-dist 图差别不大，而且它们需要的计算量要大得多。因此，我们取消了 MinPts 参数，将所有二维数据的 MinPts 设为 4。
对于二维数据）。我们提出以下交互式方法来确定 DBSCAN 的参数 Eps
的参数

### 步骤

1. 选取 k 值，建议取 k 为2*维度-1。（其中维度为特征数）
2. 计算并绘制 k-distance 图。（计算出每个点到距其第 k 近的点的距离，然后将这些距离从大到小排序后进行绘图）
3. 找到拐点位置的距离，即为 Eps 的值。
4. Minpts 的值为 k+1

![k-dist](k-distance.png)

## 实现参考

这里的参考选择了 [SimpleDBSCAN](https://github.com/CallmeNezha/SimpleDBSCAN)，一个轻量的 header-only 的 C++ 实现。SimpleDBSCAN 的实现中使用了 [kd 树](https://oi-wiki.org/ds/kdt/)来做复杂的样本划分，以加速大样本的查询。

## 最终效果

使用 DBSCAN 进行聚类的结果如下，其中 **MinPts = 10**, **Epsilon = 450**:

| 聚类结果 | 噪声 |
| ![MinPts = 10, Epsilon = 450](MinPts=10_Eps=450.png) | ![Noises](MinPts=10_Eps=450_Noises.png) |

就该结果而言，存在下面几个问题：

1. 聚类可以进一步细分，比如最上面连成一大块的紫色可以进一步划分
2. 对于比较狭窄的地方，由于采样点不足，无法形成有效聚类，从而产生噪声
3. 对于比较宽阔的地方，由于采样点之间的距离比较大，无法形成有效聚类，从而产生噪声

在引擎中导出 k-distance 数据后，使用 pyplot 绘制图像如下：

![k-distance](6dist.png)

由于样本点数目较多，因此很难直观地看出拐点在哪里，缩小一下范围，合适的值的大概位于 [250, 320] 这个区间中：

| ![slope](6dist-scale.png) | ![slope](6dist-scale2.png) |

最后的结果：

| 聚类结果 | 噪声 |
| ![MinPts = 6, Epsilon = 288](MinPts=6_Eps=288.png) | ![Noises](MinPts=6_Eps=288_Noises.png) |

## 总结



## 参考

- [scikit-leran clustering](https://scikit-learn.org/stable/modules/clustering.html#dbscan)
- [SimpleDBSCAN](https://github.com/CallmeNezha/SimpleDBSCAN)
- [DBSCAN Clustering Easily Explained with Implementation](https://www.youtube.com/watch?v=C3r7tGRe2eI)
- [DBSCAN的参数选择及其应用于离群值检测](https://blog.csdn.net/Cyrus_May/article/details/113504879)
