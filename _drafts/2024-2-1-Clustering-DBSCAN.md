---
title: DBSCAN 聚类算法
author: Yohiro
date: 2024-02-01
categories: [Algorithm, Machine Learning]
tags: [algorithm, machine learning, clustering]
math: true
render_with_liquid: false
img_path: /assets/images/DBSCAN/
image:
  path: https://scikit-learn.org/stable/_images/sphx_glr_plot_dbscan_002.png
---
## 介绍

**DBSCAN (Density-based spatial clustering of applications with noise)** 是一个基于密度的聚类算法：给定空间里的一个点集，该算法能把附近的点分成一组（有很多相邻点的点），并标记出位于低密度区域的噪声点。

原版论文在[这里](https://cdn.aaai.org/KDD/1996/KD
D96-037.pdf)

## 概念描述及参数

| 标识             | 含义                |
|:----------------|:--------------------|
| $D$             | 样本集               |
| $P_i$           | $D$ 中的各样本点      |
| $\epsilon$      | $P_i$ 的邻域距离阈值  |
| $MinPts$        | $P_i$ 在 $\epsilon$ 范围内样本的数量 |

![definition_1](Definition1.png)
_核心点与边缘点（左）和直接密度可达（右）_

核心点 (Core Points)
: 满足条件，在 $\epsilon$ 范围内，具有 $MinPts$ 个样本的样本点 $P_i$，称作核心点。

边缘点 (Border Points)
: 在**核心点** $\epsilon$ 范围内，且在样本点 $\epsilon$ 范围内的样本数量小于 $MinPts$，称作边缘点

噪声点 (Noise Points/ Outlier)
: 非可达点，即不在核心点的 $\epsilon$ 范围内的样本点，称为噪声点

直接密度可达 (directly density-reachable)
: 当 $P$ 在除 $P$ 以外的任意样本点 $Q$ 的邻域半径 $\epsilon$ 范围内，**且 $Q$ 是一个核心点**，则称 $P$ 可由 $Q$ 直接（密度）可达。

- 也就是说，核心点 $P$ 的 $\epsilon$ 范围内除 $P$ 以外的点，都是由 $P$ 直接可达的，同理，非核心点没有**直接可达**的点。

![definition_2](Definition2.png)
_可达性（左）和连接性（右）_

密度可达 (density-reachable)
: 对于样本中的点，存在路径 $P_1$,...,$P_n$，使 $P_1$ = $P$，$P_n$ = $Q$，如果该路径中任意一点 $P_{i+1}$ 可以由 $P_{i}$ 直接可达，那么称 $Q$ 是可从 $P$ （密度）可达的。

- 根据该定义，非核心点是可以由其他点可达的，但没有点是由非核心点可达的。
- 参考上图，可达性是**非对称的**，$P$ 到 $Q$ 可达不代表 $Q$ 到 $P$ 可达。

密度相连 (density-connected)
: 对于样本中的点，存在一个点 $O$ 使得 $P$ 和 $Q$ 都是由 $O$ 可达的，则点 $P$ 和点 $Q$ 被称为（密度）相连的，连接性是**对称的**。

- 同一个聚类中的任意两点都是相连的。
- 如果 $P$ 是由聚类里的点 $Q$ 可达的，那么 $P$ 和 $Q$ 在同一聚类中。

### 举例

![referenced from wikipedia](https://upload.wikimedia.org/wikipedia/commons/thumb/a/af/DBSCAN-Illustration.svg/1280px-DBSCAN-Illustration.svg.png)

上图中，$MinPts$ = 4，点 A 和其他红色点是核心点，因为它们的 $\epsilon$ 邻域（图中红色圆圈）里包含最少 4 个点（包括自己），由于它们之间相互相可达，它们形成了一个聚类。点 B 和点 C 不是核心点，但它们可由 A 经其他核心点可达，作为边缘点加入同一个聚类。点 N 是噪声点，它既不是核心点，又不由其他点可达。

### 距离函数

这里的距离可以是任意空间内的距离函数，通常使用欧氏距离，在游戏开发中也可能会使用曼哈顿距离。为直观起见，本文中的距离均指的是欧式距离。

## 算法

### 步骤

1. 初始化样本集 $D$，以及 $\epsilon$，$MinPts$ 和距离函数
2. 从任意一点 $P$ 开始探索邻域的样本点
3. 创建聚类
4. 另外的样本点出发，重复以上步骤

为确定 Cluster, DBSCAN 算法首先从集合里任意一点 $P$ 开始遍历所有密度可达点。然后根据 $\epsilon$ 和 $MinPts$ 判断 $P$ 是核心点还是边缘点。

如果 $P$ 是核心点，该流程会产生一个 Cluster。

如果 $P$ 是边缘点，那么没有任何点是从 $P$ 密度可达的，DBSCAN 将会从样本集中选择下一个点重复该过程。

对于存在多个 Cluster 的情况，算法会根据规则进行合并，这个规则是：

- Cluster 中 \$P\$ 的密度可达点属于该 Cluster
- Cluster 中 \$P\$ 的密度相连点属于该 Cluster

### 伪代码

[论文](https://cdn.aaai.org/KDD/1996/KDD96-037.pdf) 4.1 中提供了较为细致的伪代码：

```pascal
DBSCAN(SetOfPoints, Eps, MinPts)
// SetOfPoints is UNCLASSIFIED
    ClusterId := nextId(NOISE)
    FOR i FROM 1 TO SetOfPoints.size DO
        Point := SetOfPoints.get(i)
        IF Point.CiId = UNCLASSIFIED THEN
            IF ExpandCluster(SetOfPoints, Point, ClusterId, Eps, MinPts) THEN
                ClusterId := nextId(ClusterId)
            END IF
        END IF
    END FOR
END // DBSCAN
```

以及比较重要的 ExpandCluster 函数：

```pascal
ExpandCluster(SetOfPoints, Point, ClId, Eps,MinPts) : Boolean
    seeds := SetOfPoints.regionQuery(Point, Eps)
    IF seeds.size < MinPts THEN // no core point
        SetOfPoint.changeClId(Point, NOISE)
        RETURN False
    ELSE // all points in seeds are density-
         // reachable from Point
        SetOfpoints.changeClId(seeds, ClId)
        seeds .delete (Point)
        WHILE seeds <> Empty DO
            currentP := seeds.first()
            result := SetOfpoints.regionQuery(currentP, Eps)
            IF result.size >= MinPts THEN
                FOR i FROM 1 TO result.size DO
                    resultP := result.get(i)
                    IF resultP. ClId IN {UNCLASSIFIED, NOISE} THEN
                        IF resultP.ClId = UNCLASSIFIED THEN
                            seeds.append(resultP)
                        END IF
                        SetOfPoints.changeClId( resultP, ClId)
                    END IF // UNCLASSIFIED or NOISE
                END FOR
            END IF // result.size >= MinPts
            seeds.delete(currentP)
        END WHILE // seeds <> Empty
        RETURN True
    END IF
END // ExpandCluster
```

## 评估

### 稳定性

DBSCAN 的结果是确定的，对于给定顺序的数据集来说，相同参数下生成的 Clusters 是相同的。然而，当相同数据的顺序不同时，生成的 Clusters 较之另一种顺序会有所不同。

首先，即使不同顺序的数据集的核心点是相同的，Clusters 的标签会取决于数据集中各采样点的顺序。其次，可达点被分配到哪个 Clusters 也是会受到数据顺序的影响的，比如一个边缘采样点位于一个分属于两个不同的 Clusters 的核心点的 $\epsilon$ 范围内，这时该边缘点被分配到哪个 Cluster 中取决于哪一个 Cluster 先创建。

因此说 DBSCAN 是`不稳定`的。

### 效率

当 $\epsilon$ 较大，且 $D$ 数量也比较庞大时，kd 树建树的时间消耗会非常大，因此 DBSCAN `不太适合样本分布比较平均的场合`，此时可以考虑使用 [OPTICS](https://zh.wikipedia.org/wiki/OPTICS%E7%AE%97%E6%B3%95)。

## 参考

- [DBSCAN](https://zh.wikipedia.org/wiki/DBSCAN)
- [scikit-leran clustering](https://scikit-learn.org/stable/modules/clustering.html#dbscan)
- [基于密度的聚类算法（1）——DBSCAN详解](https://zhuanlan.zhihu.com/p/643338798)
- [常用聚类算法](https://zhuanlan.zhihu.com/p/104355127)
- [SimpleDBSCAN](https://github.com/CallmeNezha/SimpleDBSCAN)
- [DBSCAN Clustering Easily Explained with Implementation](https://www.youtube.com/watch?v=C3r7tGRe2eI)
- [DBSCAN的参数选择及其应用于离群值检测](https://blog.csdn.net/Cyrus_May/article/details/113504879)
