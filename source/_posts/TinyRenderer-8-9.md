---
categories: [技术,图形学,TinyRenderer]
tags: [图形学,C++,TinyRenderer,软件光栅化渲染器]
title: TinyRenderer 8&9
date: 2022-09-06 14:50:37
excerpt: TinyRendererWIki 8&9
topic: tr
---

# Lesson 8: Ambient occlusion

## 前言

在之前的学习中，我们了解了计算光照的常用方法，如Phong的光照模型。

这种光照模型是局部(Local)的——换而言之，我们在计算一个地方的光照时，并没有考虑其旁边的物体对其的影响！

在Phong模型中， 光照 = （const）Ambient + diffuse + specular

*为什么ambient要是const的呢？*

## ambient occlusion（环境光遮蔽）

与局部光照（local illumination）相对应的是全局光照（global illumination）。

显然，设定一个不变量作为环境光大小的前提条件是光线均匀而完美的照亮了每一个面，这样强的前提条件在现实中是不可能存在的，不是吗？

在过去，计算机性能还不是很好的时候，我们这么做可以节省许多性能，但是，是时候增加一点开销来换取更好的效果了！

<img src="https://s3.bmp.ovh/imgs/2022/09/06/f0487626549762c1.jpg" style="zoom:50%;" />

（这是一个*只考虑了环境光*的模型，显然，图中每个面的环境光都不一样，效果自然也不一样。）

## Brute force attempt

首先，作者为我们介绍了一种比较粗暴的方法，接下来我来简要的说明一下这种方法的思路：

1.在一个半球（hemisphere）上随机生成若干光源

（不可以仅仅是随机的选取经度和纬度：这样生成的点会集中在极点附近）

```cpp
Vec3f rand_point_on_unit_sphere() {
    float u = (float)rand()/(float)RAND_MAX;
    float v = (float)rand()/(float)RAND_MAX;
    float theta = 2.f*M_PI*u;
    float phi   = acos(2.f*v - 1.f);
    return Vec3f(sin(phi)*cos(theta), sin(phi)*sin(theta), cos(phi));
}
```

2.像之前使用过的方法一样，将eye放在光源的位置，从而得到每个光源的shadowBuffer。

我们可以使用一张TGA图来记录每次某个部分是可见还是不可见

3.重复上述过程1000次。此时我们就能得到一张完善的记录了模型每个面受到环境光照大小的纹理图！

补充：在光照图里只能看到一只手——因为模型在创作时两只手使用的是相同的纹理，因此将光照映射到纹理图时，会映射到相同的地方。

因此，对于双倍纹理坐标，我们必须进行一些修复。

这种方法较为消耗资源，适合于静态的场景进行光照的预计算。

