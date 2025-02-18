---
title: 三角函数的 FP64 近似
author: Outerra
date: 2019-11-12
categories: [Math, OpenGL]
tags: [math, optimization, opengl]
render_with_liquid: false
img_path: /assets/images/{}/
---

对 GLSL 中缺失的 FP64 函数的近似，其他的信息可以参考[**OpenGL中地图投影的双精度近似**](https://outerra.blogspot.com/2014/05/double-precision-approximations-for-map.html)

``` glsl
//sin approximation, error < 5e-9
double sina_9(double x)
{
    //minimax coefs for sin for 0..pi/2 range
    const double a3 = -1.666665709650470145824129400050267289858e-1LF;
    const double a5 =  8.333017291562218127986291618761571373087e-3LF;
    const double a7 = -1.980661520135080504411629636078917643846e-4LF;
    const double a9 =  2.600054767890361277123254766503271638682e-6LF;

    const double m_2_pi = 0.636619772367581343076LF;
    const double m_pi_2 = 1.57079632679489661923LF;

    double y = abs(x * m_2_pi);
    double q = floor(y);
    int quadrant = int(q);

    double t = (quadrant & 1) != 0 ? 1 - y + q : y - q;
    t *= m_pi_2;

    double t2 = t * t;
    double r = fma(fma(fma(fma(a9, t2, a7), t2, a5), t2, a3), t2*t, t);

    r = x < 0 ? -r : r;

    return (quadrant & 2) != 0 ? -r : r;
}

//sin approximation, error < 2e-11
double sina_11(double x)
{
    //minimax coefs for sin for 0..pi/2 range
    const double a3 = -1.666666660646699151540776973346659104119e-1LF;
    const double a5 =  8.333330495671426021718370503012583606364e-3LF;
    const double a7 = -1.984080403919620610590106573736892971297e-4LF;
    const double a9 =  2.752261885409148183683678902130857814965e-6LF;
    const double ab = -2.384669400943475552559273983214582409441e-8LF;

    const double m_2_pi = 0.636619772367581343076LF;
    const double m_pi_2 = 1.57079632679489661923LF;

    double y = abs(x * m_2_pi);
    double q = floor(y);
    int quadrant = int(q);

    double t = (quadrant & 1) != 0 ? 1 - y + q : y - q;
    t *= m_pi_2;

    double t2 = t * t;
    double r = fma(fma(fma(fma(fma(ab, t2, a9), t2, a7), t2, a5), t2, a3),
        t2*t, t);

    r = x < 0 ? -r : r;

    return (quadrant & 2) != 0 ? -r : r;
}
```

对于 Cosine 而言，平移 π/2 即可

``` glsl
//cos approximation, error < 5e-9
double cosa_9(double x)
{
    //sin(x + PI/2) = cos(x)
    return sina_9(x + DBL_LIT(1.57079632679489661923LF));
}

//cos approximation, error < 2e-11
double cosa_11(double x)
{
    //sin(x + PI/2) = cos(x)
    return sina_11(x + DBL_LIT(1.57079632679489661923LF));
}
```
