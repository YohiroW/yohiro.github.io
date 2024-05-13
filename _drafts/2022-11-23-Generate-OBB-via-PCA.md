---
title: 使用 PCA 方法创建有向包围盒（OBB）
author: Yohiro
date: 2022-11-23
categories: [Engine, Algorithm]
tags: [geomertry, OBB, AABB]
render_with_liquid: false
math: true
img_path: /assets/images/OBB/
---

## 背景

在游戏开发中，常见的包围盒有*球形包围盒*、*轴对齐包围盒（AABB）*、*有向包围盒（OBB）*、*K-DOP* 以及*凸包*等，包围盒通常用于判断可见性以及碰撞检测等领域。其中，AABB 和 OBB 的应用最为广泛。

![bounding-box](bounding-box.png)
_不同类型的包围盒_

球形包围盒
: 球形包围盒是包含其所属对象的最小球体。一般用于较粗粒度的碰撞检测。

AABB (**A**xis-**A**ligned **B**ounding **B**ox)
: 轴对齐包围盒（AABB） 是包含其所属对象的且各边平行于各坐标轴的最小六面体。它不会相对于轴进行旋转，因此可以仅通过 Min 和 Max 来进行定义，实现以及计算非常简单。但也由于不进行旋转，所以空间冗余很大。

OBB (**O**riented **B**ounding **B**ox)
: 定向包围盒（OBB）是包含其所属对象的且相对于坐标轴方向任意的最小的长方体。相较于 AABB，OBB 的空间利用率更高，但求交的时候也会变得复杂。
    体

K-DOP (**D**iscrete **O**riented **P**olytope)
: 离散定向多面体 （DOP）是包含其所属对象的二维空间的凸多边形或者三维空间的凸多面体，它是一组无限远的定向平面移动到与物体相交而得到，于是 DOP 就是这些平面相交平面所生成的凸多面体。从 K 个平面构建的 DOP 称为 k-DOP。

凸包 (Convex Hull)
: 凸包是包容物体的最小凸体。如果物体是有限个点的集合，那么凸包就是一个多面体，实际上它是包容多面体的最小立体。

这篇文章主要用于阐述如何使用主成分分析（**P**rincipal **C**omponents **A**nalysis）方法生成 OBB。

## PCA 介绍

主成分分析是一种统计分析、简化数据集的方法。它利用正交变换来对一系列可能相关的变量的观测值进行线性变换，从而投影为一系列线性不相关变量的值，这些*不相关变量*称为*主成分*。简单来讲，PCA 解决了这样一个问题：

> 如果我们有一组N维向量，现在要将其降到K维（K小于N），那么我们应该如何选择K个基才能最大程度保留原有的信息？

想象

这里来回顾一下数理统计和线性代数的内容：

在统计学里，协方差（Covariance）用于描述数据的相关程度。协方差越大，样本的相关性越强。

![Covariance Trends](Covariance_trends.png){: .right}

对期望为 $E(X)$ 和 $E(Y)$ 的随机变量 $X$ 和 $Y$ 的协方差定义如下：

$$\begin{equation}
cov(X,Y) = E[(X-E(X))(Y-E(Y))]
\end{equation}$$

从上式可知，$cov(X,Y) = cov(Y,X)$。  

参考右图，当 $cov(X,Y) > 0$ 时，两变量*线性正相关*；当 $cov(X,Y) < 0$ 时，两变量*线性负相关*。
而当 $cov(X,Y) == 0$ 时，两变量*线性无关*。

假设我们的数据是三维的，通过计算各维度数据的协方差后，可以得到这样的一个*实对称矩阵*，也就是**协方差矩阵**：

$$\begin{bmatrix}
cov(X,X) & cov(X,Y) & cov(X,Z) \\
cov(Y,X) & cov(Y,Y) & cov(Y,Z) \\
cov(Z,X) & cov(Z,Y) & cov(Z,Z) \\
\end{bmatrix}$$

主对角线的元素是变量与其自身的协方差，代表该变量的方差。

按照 PCA 的定义，不相关变量是主成分，不相关的变量的协方差等于零，而在协方差矩阵的非对角线位置的是各维度变量的协方差。因此，我们需要对协方差矩阵做变换，使非对角线上的元素化为零。
也就是将协方差矩阵**对角化**。

通过之前图形学的学习，我们知道将本地坐标系的某一点 P 转换到观察坐标系下需要乘观察矩阵，也就是矩阵乘法定义了一组变换：

$$\begin{equation}
P_V = ViewMatrix * P
\end{equation}$$

其中 P 为列向量，ViewMatrix 为观察矩阵。

![PCA](PCA.png){: .w-50}
_一个高斯分布的主成分分析。黑色的两个向量是此分布的协方差矩阵的特征向量，其长度为对应的特征值之平方根，并以分布的平均值为原点。_

## 算法



### 代码

这里使用 [eigen](https://github.com/PX4/eigen) 来做矩阵计算以及求解特征向量：

```cpp
static void ComputePCA(const TArray<FVector3f>& InPoints, const uint32 InPointOffset, const uint32 InPointCount, FVector3f& Mean, FVector3f& B0, FVector3f& B1, FVector3f& B2)
{
    Eigen::Matrix3f covariance = ComputeCovarianceMatrix(InPoints, Mean);
    Eigen::SelfAdjointEigenSolver<Eigen::Matrix3f> eigen_solver(covariance, Eigen::ComputeEigenvectors);
    Eigen::Matrix3f eigenVectorsPCA = eigen_solver.eigenvectors();

    B0 = FVector3f(eigenVectorsPCA.col(0)(0), eigenVectorsPCA.col(0)(1), eigenVectorsPCA.col(0)(2));
    B1 = FVector3f(eigenVectorsPCA.col(1)(0), eigenVectorsPCA.col(1)(1), eigenVectorsPCA.col(1)(2));
    B2 = FVector3f(eigenVectorsPCA.col(2)(0), eigenVectorsPCA.col(2)(1), eigenVectorsPCA.col(2)(2));

    B0 = B0.GetSafeNormal();
    B1 = B1.GetSafeNormal();
    B2 = B2.GetSafeNormal();

    FVector3f LocalMinBound(FLT_MAX);
    FVector3f LocalMaxBound(-FLT_MAX);
    //for (const FVector3f& P : InPoints)
    for (uint32 PointIt = 0; PointIt < InPointCount; ++PointIt)
    {
        FVector3f PLocal = InPoints[InPointOffset + PointIt] - Mean;
        PLocal = FVector3f(
            FVector3f::DotProduct(PLocal, B0),
            FVector3f::DotProduct(PLocal, B1),
            FVector3f::DotProduct(PLocal, B2));

        LocalMinBound.X = FMath::Min(LocalMinBound.X, PLocal.X);
        LocalMinBound.Y = FMath::Min(LocalMinBound.Y, PLocal.Y);
        LocalMinBound.Z = FMath::Min(LocalMinBound.Z, PLocal.Z);

        LocalMaxBound.X = FMath::Max(LocalMaxBound.X, PLocal.X);
        LocalMaxBound.Y = FMath::Max(LocalMaxBound.Y, PLocal.Y);
        LocalMaxBound.Z = FMath::Max(LocalMaxBound.Z, PLocal.Z);
    }

    // Recompte the new mean in local space, and then compute its world position
    const FVector3f LocalMean(
        (LocalMaxBound.X + LocalMinBound.X) * 0.5f,
        (LocalMaxBound.Y + LocalMinBound.Y) * 0.5f,
        (LocalMaxBound.Z + LocalMinBound.Z) * 0.5f);

    const FVector3f LocalExtent(
        (LocalMaxBound.X - LocalMinBound.X) * 0.5f,
        (LocalMaxBound.Y - LocalMinBound.Y) * 0.5f,
        (LocalMaxBound.Z - LocalMinBound.Z) * 0.5f);

    const FVector3f MeanPrime = Mean + B0 * LocalMean.X + B1 * LocalMean.Y + B2 * LocalMean.Z;
    Mean = MeanPrime;

    B0 *= LocalExtent.X;
    B1 *= LocalExtent.Y;
    B2 *= LocalExtent.Z;
}
```

## 参考

- [包围体](https://zh.wikipedia.org/wiki/%E5%8C%85%E5%9B%B4%E4%BD%93)
- [主成分分析](https://zh.wikipedia.org/wiki/%E4%B8%BB%E6%88%90%E5%88%86%E5%88%86%E6%9E%90)
- [Covariance matrix](https://en.wikipedia.org/wiki/Covariance_matrix)
- [PCA的数学原理](http://blog.codinglabs.org/articles/pca-tutorial.html)
- [【引擎研发】OBB生成与可视化](https://zhuanlan.zhihu.com/p/523291781)
- [OBB generation via Principal Component Analysis](https://hewjunwei.wordpress.com/2013/01/26/obb-generation-via-principal-component-analysis/)
- [PCA主成分分析学习总结](https://zhuanlan.zhihu.com/p/32412043)
