---
title: TinyRenderer 0&1
excerpt: TinyRendererWIki 0&1
date: 2022-05-05 15:59:04
categories: [技术, 图形学]
tags: [TinyRenderer, C++, 软件光栅化渲染器]
topic: tr
---
# 简介

TinyRenderer是国外一个大佬写的几百行的软件光栅化渲染器，API模拟了OpenGL，主要目的在于理解OpenGL在后台干了什么。

通过复现这段大约500行的代码, 我想我能更加深入的了解图形学的底层原理.

有些话适合用中文,有些话适合用英文(包括一些引用),原谅我中英混杂(反正也不算是教程吧

(btw, I want to say that this is not just a simple translated version of the wiki on github, it is consist of my own feelings and thoughts.)

我的IDE(雾)使用的是VScode搭配自带的插件, 多文件编译直接使用了自带的插件C/C++ Project Generator.

顺带一提，里面贴的代码都是我自己写的，可能不够工整和正确，如果需要能保证规范正确的源代码请访问[该项目的GitHub](https://github.com/ssloy/tinyrenderer/wiki)。

# TinyRenderer 0 : draw a ponit on your screen

{% blockquote %}

Since the goal is to minimize external dependencies, I am allowed to only work with [TGA](http://en.wikipedia.org/wiki/Truevision_TGA) files. It’s one of the simplest formats that supports images in RGB/RGBA/black and white formats. So, as a starting point, I’ll obtain a simple way to work with pictures. You should note that the only functionality available at the very beginning (in addition to loading and saving images) is the capability to set the color of one pixel.

{% endblockquote %}

There are no functions for drawing line segments and triangles. I’ll have to do all of this by hand.

## Include tgas file

因为开始这个项目需要对tga文件的读取和保存功能，我们需要先下载 `tgaimage.h`和 `tgaimage.cpp`两个文件，并放入我们的项目中。引入后，可能需要先熟悉一下自带的方法，以备之后使用所需。

此外，我为了在电脑上直接查看tga类型的文件，还在网上下载了一个tga view软件。

## just draw a point

因为这一步太简单，我没什么好说的。

直接复制啦~

```cpp
#include "tgaimage.h"
const TGAColor white = TGAColor(255, 255, 255, 255);
const TGAColor red   = TGAColor(255, 0,   0,   255);
int main(int argc, char** argv) {
        TGAImage image(100, 100, TGAImage::RGB);
        image.set(52, 41, red);
        image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
        image.write_tga_file("output.tga");
        return 0;
}
```

# TinyRenderer 1 : draw a line

在本章节中,作者总共进行了5次尝试:

## 1. 画一条普通的线

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (float t=0.; t<1.; t+=.01) { 
        int x = x0 + (x1-x0)*t; 
        int y = y0 + (y1-y0)*t; 
        image.set(x, y, color); 
    } 
}
```

这种算法固然可以,但是由于我们很难决定t的增量值:

当t的增量过小,产生大量资源浪费

当t的增量过大,线条割裂感严重
因此这是一种低效的算法

进行改良:

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (int x=x0; x<=x1; x++) { 
        float t = (x-x0)/(float)(x1-x0); 
        int y = y0*(1.-t) + y1*t; 
        image.set(x, y, color); 
    } 
}
```

此时,t的大小自动产生,不需要我们自己设置了!但是产生了新的问题:

## 2.同时画三条线,各自具备自己的特点:

```
line(13, 20, 80, 40, image, white); 
line(20, 13, 40, 80, image, red); 
line(80, 40, 13, 20, image, red);
```

![](https://s3.bmp.ovh/imgs/2022/05/14/31d795e4522403f6.png)

此时可以看到, 结果和我们预想的明显不同, 那么发生了什么问题呢?

第一条线, 和第一次尝试一样没有区别, 成功

第二条线是一条陡峭(steep\)的线,此时我们可以看到线段非常明显的产生割裂--->x的增量小,y的增量大,明显,应该考虑利用y生成x

第三条线完全没出现: 因为是从右到左给的点,函数不兼容!

有没有什么办法,可以使其具有对称性(交换两点的位置得到相同的直线)和不受线段斜率影响呢?

## 3.进行算法的改良

```cpp
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { // if the line is steep, we transpose the image 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { // make it left−to−right 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    for (int x=x0; x<=x1; x++) { 
        float t = (x-x0)/(float)(x1-x0); 
        int y = y0*(1.-t) + y1*t; 
        if (steep) { 
            image.set(y, x, color); // if transposed, de−transpose 
        } else { 
            image.set(x, y, color); 
        } 
    } 
}
```

可以看到,此时我们解决了以上两种情况,但是,到此为止了吗?

图形学是一个性能敏感的学科!尤其是实时渲染, 非常看重fps,因此,我们尝试着优化一下性能吧?

不急,先看看哪部分最值得优化:

```
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 69.16      2.95     2.95  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 19.46      3.78     0.83 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
  8.91      4.16     0.38 207000000     0.00     0.00  TGAColor::TGAColor(TGAColor const&) 
  1.64      4.23     0.07        2    35.04    35.04  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
  0.94      4.27     0.04                             TGAImage::get(int, int)
```

可以看到, 画线段的line函数花费了70%的时间!(果然是我们函数写的太丑了

下面我们来看看看如何优化吧!

## 4.算法的性能优化

先上代码!

```
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    int dx = x1-x0; 
    int dy = y1-y0; 
    float derror = std::abs(dy/float(dx)); 
    float error = 0; 
    int y = y0; 
    for (int x=x0; x<=x1; x++) { 
        if (steep) { 
            image.set(y, x, color); 
        } else { 
            image.set(x, y, color); 
        } 
        error += derror; 
        if (error>.5) { 
            y += (y1>y0?1:-1); 
            error -= 1.; 
        } 
    } 
} 
```

区别在哪里?

在最新的函数里,我们将dx,dy提出来做了计算,这样就不用在循环里做反复的计算了!可以想象,应该能省小一半的时间吧!

```
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 38.79      0.93     0.93  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 37.54      1.83     0.90 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
 19.60      2.30     0.47 204000000     0.00     0.00  TGAColor::TGAColor(int, int) 
  2.09      2.35     0.05        2    25.03    25.03  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
  1.25      2.38     0.03                             TGAImage::get(int, int) 
```

果不其然!省了一大半的时间!

还能不能继续省呢?

可以! C++语言中, 浮点数计算时间远超整型数!

既然像素一定是整数,为什么我们要用浮点数!不如取整吧!

## 5.Bresenham's Line Drawing Algorithm

```
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    int dx = x1-x0; 
    int dy = y1-y0; 
    int derror2 = std::abs(dy)*2; 
    int error2 = 0; 
    int y = y0; 
    for (int x=x0; x<=x1; x++) { 
        if (steep) { 
            image.set(y, x, color); 
        } else { 
            image.set(x, y, color); 
        } 
        error2 += derror2; 
        if (error2 > dx) { 
            y += (y1>y0?1:-1); 
            error2 -= dx*2; 
        } 
    } 
} 
```

嗯嗯, 摆脱了浮点数,再来看看时间:

```
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 42.77      0.91     0.91 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
 30.08      1.55     0.64  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 21.62      2.01     0.46 204000000     0.00     0.00  TGAColor::TGAColor(int, int) 
  1.88      2.05     0.04        2    20.02    20.02  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
```

又快了不少! 甚至掉下耗时第一名了!真厉害!

## 6.总结

结合五次尝试,我说说我自己的感想吧:

1. 光栅化是以屏幕为基础的,也就是说心里一定要有像素的概念! 现实世界不能无限细分,有些数学会被限制拳脚,但也因此我们能产生像bresenham算法这样化零为整的偷懒方法!
2. 性能优化比想象的重要. 图形学的算法几乎都是极度复杂的递归或者迭代算法,一个小小的改变就能极大影响速度
   通过两个简单的改动,我们将line()函数优化到了原来的1/5.
3. 考虑一定要周全. 图形学是一个不在乎边界条件的学科,但是也是一个非常严谨的学科. 一个函数的不严谨会在无数次调用中发挥巨大的杀伤力

**这里我必须告诫我自己: 你一定要能动手写代码!千万不能照抄! 一定要能自己写! 看着简单但其实不写是不会发现坑的!**

最后贴一下自己的代码:

```cpp
//bresenham's line drawing algorithem
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) {
    bool steep = false;
    if (std::abs(x0-x1)<std::abs(y0-y1)) {
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if (x0>x1) {
        std::swap(x0, x1);
        std::swap(y0, y1);
    }
    int dy = y1 - y0;
    int dx = x1 - x0;
    int error = 0;
    int derror = std::abs(dy) * 2;
    for (int x = x0,y = y0; x <= x1; x++)
    {
  
        if (steep) {
            image.set(y, x, color);
        } else {
            image.set(x, y, color);
        }
        error += derror;
        if(errno>dx)
        {
            y += (y1>y0?1:-1); 
            error -= dx*2; 
        }
    }
}
```
