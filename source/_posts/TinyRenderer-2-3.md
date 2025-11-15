---
title: TinyRenderer 2&3
excerpt: TinyRendererWIki 2&3
date: 2022-05-14 15:59:04
categories: [技术, 图形学]
tags: [TinyRenderer, C++, 软件光栅化渲染器]
topic: tr
---
# Lesson 2: Triangle rasterization and back face culling

## 1.尝试画一个三角形(old-school)

画一个三角形看似简单,但是又很难!

先想想如何画一个三角形(不要忘了光栅化的前提:像素的概念)

首先, 画一个三角形框吧!

```cpp
void triangle(Vec2i v0, Vec2i v1, Vec2i v2, TGAImage &image, TGAColor color)
{
	line(v0.x, v0.y, v1.x, v1.y, image, color);
	line(v1.x, v1.y, v2.x, v2.y, image, color);
	line(v0.x, v0.y, v2.x, v2.y, image, color);
}
```

很简单!

在主函数里调用此方法:

```cpp
int main(int argc, char** argv) {
	TGAImage image = TGAImage(width, height, TGAImage::RGB);
	Vec2i t0[3] = {Vec2i(10, 70),   Vec2i(50, 160),  Vec2i(70, 80)}; 
	Vec2i t1[3] = {Vec2i(180, 50),  Vec2i(150, 1),   Vec2i(70, 180)}; 
	Vec2i t2[3] = {Vec2i(180, 150), Vec2i(120, 160), Vec2i(130, 180)}; 
	triangle(t0[0], t0[1], t0[2], image, red); 
	triangle(t1[0], t1[1], t1[2], image, white); 
	triangle(t2[0], t2[1], t2[2], image, green);
	image.flip_vertically(); 
    image.write_tga_file("output.tga");
	return 0;
}
```

我们得到想要的图形:

![](https://s3.bmp.ovh/imgs/2022/05/14/d4f5acfb711300b2.png)

接着,想办法填充吧!

提醒:将三角形水平分割.

{% blockquote %}

无论是原作者还是笔者, 都强烈建议您在接着往下看之前先自己尝试实现此功能! 自己实在写不出来或者写出来了再往下看!

{% endblockquote %}

贴一份我自己的代码吧(比较稚嫩)

```cpp
/
/draw a triangle use line the function
void triangle(Vec2i v0, Vec2i v1, Vec2i v2, TGAImage &image, TGAColor color)
{
	//排序
    if(v0.y>v1.y) 
    {
        Vec2i temp = v0;
        v0 = v1;
        v1 = temp;
    }
    if(v1.y>v2.y)
    {
        Vec2i temp = v1;
        v1 = v2;
        v2= temp;
    }
    if(v0.y>v1.y)
    {
        Vec2i temp = v0;
        v0 = v1;
        v1 = temp;
    }
    float x3 = (float)v0.y + (float)(v1.y - v0.y) * (v2.x - v0.x) / (v2.y - v0.y);
    float dx1 = (float)(v2.x - v0.x) / (v2.y - v0.y);
    float dx2 = (float)(v1.x - v0.x) / (v1.y - v0.y);
    //printf("dx1:%f dx2:%f\n", dx1, dx2);
    if (dx1 > dx2)
        std::swap(dx1, dx2);
    float xleft = (float)v0.x, xright = (float)v0.x;
    for (int y = v0.y; y < v1.y; y++)
    {
        for (int x = std::ceil(xleft); x <= std::floor(xright); x++)
        {
            image.set(x, y, color);
        }
        xleft += dx1;
        xright += dx2;
    }
    dx1 = (float)(v2.x - v0.x) / (v2.y - v0.y);
    dx2 = (float)(v2.x - v1.x) / (v2.y - v1.y);
    if (dx1 > dx2)
        std::swap(dx1, dx2);
    xleft = (float)v2.x, xright = (float)v2.x;
    //printf("dx1:%f dx2:%f\n", dx1, dx2);
    for (int y = v2.y; y >= v1.y; y--)
    {
         //printf("%f %f\n", xleft, xright);
        for (int x = std::ceil(xleft); x <= std::floor(xright); x++)
        {
            image.set(x, y, color);
        }
        xleft -= dx2;
        xright -= dx1;
    }
}
```

再贴一份大神的研读一下,我自己写了一些注释

```cpp
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    if (t0.y==t1.y && t0.y==t2.y) return; // I dont care about degenerate triangles 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    // 排序! 写的比我简洁很多! 我都不知道原来vec2i也可以swap
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    //降低可读性换取代码的整洁和易修改, 可以的
    //大神的填充三角形的方法和我想的差不多,对于每个y找到x的左右边界  
    //大神没有考虑边界的问题是否取的问题, 效果也不错
    for (int i=0; i<total_height; i++) { 
        bool second_half = i>t1.y-t0.y || t1.y==t0.y; 
        int segment_height = second_half ? t2.y-t1.y : t1.y-t0.y; 
        float alpha = (float)i/total_height; 
        float beta  = (float)(i-(second_half ? t1.y-t0.y : 0))/segment_height; // be careful: with above conditions no division by zero here 
        Vec2i A =               t0 + (t2-t0)*alpha; 
        Vec2i B = second_half ? t1 + (t2-t1)*beta : t0 + (t1-t0)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, t0.y+i, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
}
```

## 2.新潮一些的方法

上面介绍的方法是一种非常传统和老旧的方法(当时是在单线程电脑上运行),其实随着电脑硬件的发展,目前即使是民用PC也支持超多线程的运算了.这样写代码,效率是低下的.不如我们来想想怎么写效率才高?

至少对于笔者而言,我也不知道怎么才能针对多线程进行优化, 代码运行的底层原理知识笔者严重缺失,因此直接贴出来:

```cpp
triangle(vec2 points[3]) { 
    vec2 bbox[2] = find_bounding_box(points); 
    for (each pixel in the bounding box) { 
        if (inside(points, pixel)) { 
            put_pixel(pixel); 
        } 
    } 
}
```

在笔者熟悉的算法竞赛和课程中,这样写的效率是非常低的.

但是大神说这是一种针对多线程更优的写法,那笔者就先记住吧,等以后再来想为什么.

其实这个伪代码对于上过games101的人来说应该在熟悉不过了.

一起来看看这段伪代码:

首先,我们找到三角形的包围盒(这是很简单的)

然后,我们对于盒内每个点判断其是否在三角形内

这是很复杂的.有很多方法,但是最主流的需要用到重心坐标的概念

$P = \alpha A + \beta B + \gamma C $

若P在三角形外,则必有坐标为负

贴一下我自己的实现

```cpp
Vec3f barycentric(Vec2i *pts, Vec2i P) {
    Vec3f u1 = Vec3f(pts[2].x - pts[0].x, pts[1].x - pts[0].x, pts[0].x - P.x);
    Vec3f u2 = Vec3f(pts[2].y - pts[0].y, pts[1].y - pts[0].y, pts[0].y - P.y);
    Vec3f u = Vec3f(u1.y * u2.z - u1.z * u2.y, u1.z * u2.x - u1.x * u2.z, u1.x * u2.y - u1.y * u2.x);
    if (std::abs(u.z) < 1)
        return Vec3f(-1, 1, 1);
    return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z); 
} 

//draw a triangle use a new way
void new_triangle(Vec2i *pts, TGAImage &image, TGAColor color)
{
    Vec2i bboxmin = Vec2i(image.get_width() - 1, image.get_height() - 1);
    Vec2i bboxmax = Vec2i(0, 0);
    for (int i = 0; i < 3; i++)
    {
        if(pts[i].x>bboxmax.x)
            bboxmax.x = pts[i].x;
        if(pts[i].y>bboxmax.y)
            bboxmax.y = pts[i].y;
        if(pts[i].x<bboxmin.x)
            bboxmin.x = pts[i].x;
        if(pts[i].y<bboxmin.y)
            bboxmin.y = pts[i].y;
    }
    for (int x = bboxmin.x; x <= bboxmax.x; x++)
    {
        for (int y = bboxmin.y; y <= bboxmax.y; y++)
        {
            Vec3f coor = barycentric(pts, Vec2i(x, y));
            //printf("%f %f %f\n", coor.x, coor.y, coor.z);
            if (coor.x >= 0 && coor.y >= 0 && coor.z >= 0)
            {
                //puts("set");
                image.set(x, y, color);
            }
        }
    }
}
```

## 3.试着用这种方法填充之前的模型

还记得在bresenham的画线的章节里我们使用的模型吗?

现在我们有能力为他上色啦!

先用随机颜色上色,这样我们可以观察渲染是如何进行的,然后,再根据光照上色,这样我们就完成了一个简单的渲染器!

先使用随机的颜色吧!

```cpp
//main函数
if (2==argc) {
        model = new Model(argv[1]);
    } else {
        model = new Model("obj/african_head.obj");
    }
    TGAImage frame(width, height, TGAImage::RGB); 

    for (int i=0; i<model->nfaces(); i++) 
    { 
        std::vector<int> face = model->face(i); 
        Vec2i screen_coords[3]; 
        for (int j=0; j<3; j++) { 
            Vec3f world_coords = model->vert(face[j]); 
            screen_coords[j] = Vec2i((world_coords.x+1.)*width/2., (world_coords.y+1.)*height/2.); 
        }
        Vec2i pts[3] = {screen_coords[0], screen_coords[1], screen_coords[2]};
        new_triangle(pts, frame, TGAColor(rand() % 255, rand() % 255, rand() % 255, 255));
    }
    frame.flip_vertically(); // to place the origin in the bottom left corner of the image 
    frame.write_tga_file("framebuffer.tga");
    return 0;
```

我们可以看到, model里面储存的是世界坐标, 是一群被压缩在[-1,1]^2的点. 为了使其铺满屏幕,我们需要进行一定的转化.

那么, 如何结合光照呢? 在讨论布林冯等模型之前,我们不如先进行简单的线性建模

先确立光线的方向 `Vec3f light_dir(0,0,-1)`,此处明显是平行光.

那不妨直接将每个面的亮暗设置为 `n*light_dir`, n为一个面的单位法向量.

下面为我的代码:

```cpp
Vec3f light_dir(0, 0, -1);
    for (int i = 0; i < model->nfaces(); i++)
    { 
        std::vector<int> face = model->face(i); 
        Vec2i screen_coords[3];
        Vec3f world_coords[3];
        for (int j = 0; j < 3; j++)
        {
            world_coords[j] = model->vert(face[j]); 
            screen_coords[j] = Vec2i((world_coords[j].x+1.)*width/2., (world_coords[j].y+1.)*height/2.);
        }
        Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]);
        n.normalize();
        float intensity = n * light_dir;
	Vec2i pts[3] = {screen_coords[0], screen_coords[1], screen_coords[2]};
        new_triangle(pts, frame, TGAColor(intensity * 255, intensity * 255, intensity * 255, 255));
    }
    frame.flip_vertically(); // to place the origin in the bottom left corner of the image 
    frame.write_tga_file("framebuffer.tga");
    return 0;
```

# Lesson 3: Hidden faces removal (z buffer)

## 1.zBuffer储存深度

现在我们已经可以渲染一个面啦!很好!

但是还有一个问题没解决----不同面的遮挡关系该如何处理?

在原来代码中, 越晚渲染的面在越前面, 这固然有一定的道理, 但是这不是现实生活

我们要理解光栅化其实是把三维空间里的模型二维化, 然后以一定的比例投影到屏幕上的过程, 那么, 任何一个点一定有他的深度----也就是z轴的大小.

![](https://s3.bmp.ovh/imgs/2022/05/14/6a05ca12f47aff17.png)

看看这个图, 如果能成功渲染这个画面, 是不是有点三维的感觉了?

使用Z-buffer数组来储存, 通过重心坐标来计算每点P的z值, 并更新z-buffer

下面贴一下我的改良后的triangle代码:

```cpp
void new_triangle(Vec3f *pts, float *zBuffer, TGAImage &image, TGAColor color)
{
    Vec2i bboxmin = Vec2i(image.get_width() - 1, image.get_height() - 1);
    Vec2i bboxmax = Vec2i(0, 0);
    for (int i = 0; i < 3; i++)
    {
        if(pts[i].x>bboxmax.x)
            bboxmax.x = pts[i].x;
        if(pts[i].y>bboxmax.y)
            bboxmax.y = pts[i].y;
        if(pts[i].x<bboxmin.x)
            bboxmin.x = pts[i].x;
        if(pts[i].y<bboxmin.y)
            bboxmin.y = pts[i].y;
    }
    for (int x = bboxmin.x; x <= bboxmax.x; x++)
    {
        for (int y = bboxmin.y; y <= bboxmax.y; y++)
        {
            Vec3f coor = barycentric(pts, Vec2i(x, y));
            if (coor.x >= 0 && coor.y >= 0 && coor.z >= 0)
            {
                float z = coor.x * pts[0].z + coor.y * pts[1].z + coor.z * pts[2].z;
                if (zBuffer[x + y * width] < z)
                {
                    zBuffer[x + y * width] = z;
                    image.set(x, y, color);
                }
            }
        }
    }
}
```

干脆把主函数也贴一下, 以免有人不知道该怎么获取深度

```cpp
    float zBuffer[width * height];//创建zbuffer数组
    memset(zBuffer, -10, sizeof(zBuffer));//初始化zbuffer数组
					  //此处需要注意,在obj文件中,点的深度均处于[-1,0],如果不进行初始化或者把z转换为负数,会显示错误
    for (int i = 0; i < model->nfaces(); i++)
    { 
        std::vector<int> face = model->face(i); 
        Vec3f screen_coords[3];
        Vec3f world_coords[3];
        TGAColor color;
        for (int j = 0; j < 3; j++)
        {
            world_coords[j] = model->vert(face[j]);   
            screen_coords[j] = Vec3f((world_coords[j].x+1.)*width/2., (world_coords[j].y+1.)*height/2., world_coords[j].z);
        }
        Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]);
        n.normalize();
        float intensity = n * light_dir;
	    Vec3f pts[3] = {screen_coords[0], screen_coords[1], screen_coords[2]};
        if(intensity>0)//别忘了,很重要
            new_triangle(pts, zBuffer, frame, TGAColor(intensity * 255, intensity * 255, intensity * 255, 255));
    }
```

## 2.填充纹理吧

此时, 一个模型大体的轮廓已经初步创建, 那么 ,我们还能做什么呢???

没错! 填充纹理!!

在obj文件中, 除了存放了模型本身的面和点, 也存放了每个点对应的纹理坐标. 通过这些映射关系, 我们对于每个特定的像素(x,y)都可以在纹理图上映射为(u,v)并获取该点的颜色为该像素上色.

```cpp
vt  0.532 0.923 0.000 //存放了第i个纹理图上的点
f 24/1/24 25/2/25 26/3/26 //存放了某个面的对应信息
//格式："f 顶点索引/uv点索引/法线索引"
```

想要完成这个作业有两个难点,一是读取纹理信息, 二是对纹理插值

其实我觉得老师对于这个地方的处理不算是很好, 因为没有告诉你应该怎么去做, 而是想让你凭空去创造obj的读取方法.(不排除这就是老师的本意!

既然如此,那就硬着头皮上呗!

理解obj文件的解析原理是关键, 如果能理解, 这个代码其实并不难, 就是普通的模拟题.

obj文件里面还每个点的法线信息, 干脆也一起敲了

下面分享一下我的model.h和model.cpp

```cpp
//model.h
#ifndef __MODEL_H__
#define __MODEL_H__

#include <vector>
#include "geometry.h"
#include "tgaimage.h"

class Model {
private:
std::vector<Vec3f> verts_;
	//添加储存纹理坐标和法线的方法
	std::vector<Vec2f> uvs_; 
	std::vector<Vec3f> nrms_;
	std::vector<int> faces_vet;
	std::vector<int> faces_uvs;
	std::vector<int> faces_nrm;
	void load_texture(const std::string *filename, TGAImage &texture);
public:
	Model(const char *filename);
	~Model();
	int nverts();
	int nfaces();
	int nuvs();
	int nnrms();
	Vec3f vert(int i);
	Vec2f uvs(int i);
	Vec3f nrms(int i);
	Vec2f face_uvs(int facei, int idx);
	Vec3f face_vet(int facei, int idx);
	Vec3f face_nrm(int facei, int idx);
};

#endif //__MODEL_H__

```

```cpp
#include <iostream>
#include <string>
#include <fstream>
#include <sstream>
#include <vector>
#include "model.h"

Model::Model(const char *filename){
    std::ifstream in;
    in.open (filename, std::ifstream::in);
    if (in.fail()) return;
    std::string line;
    while (!in.eof()) {
        std::getline(in, line);
        std::istringstream iss(line.c_str());
        char trash;
        if (!line.compare(0, 2, "v ")) {
            iss >> trash;
            Vec3f v;
            for (int i=0;i<3;i++) iss >> v.raw[i];
            verts_.push_back(v);
        } else if(!line.compare(0, 3, "vt "))
        {
            iss >> trash >> trash;
            Vec2f uv;
            for (int i=0;i<2;i++) iss >> uv.raw[i];
            uvs_.push_back({uv.x, 1 - uv.y});//非常需要注意! 纹理图是以左上为原点的,而模型文件里的储存是以左下为原点, 需要颠倒一下!
        }
        else if (!line.compare(0, 3, "vn "))
        {
            iss >> trash >> trash;
            Vec3f n;
            for (int i=0;i<3;i++) iss >> n.raw[i];
            nrms_.push_back(n);
        }
        else if (!line.compare(0, 2, "f ")) {
            int idx,tidx,nidx;
            iss >> trash;
            while (iss >> idx >> trash >> tidx >> trash >> nidx) {
                faces_nrm.push_back(--nidx);
                faces_uvs.push_back(--tidx);
                faces_vet.push_back(--idx);
            }
        }
    }
    //std::cerr << "# v# " << verts_.size() << " f# "  << faces_.size() << std::endl;//报错信息,我不会先不管
}

void load_texture(const std::string filename, TGAImage &texture)
{
    texture.read_tga_file(filename.c_str());
}

Model::~Model() {
}

int Model::nverts() {
    return (int)verts_.size();
}

int Model::nfaces() {
    return (int)(faces_vet.size() / 3);
}

int Model::nuvs() {
    return (int)uvs_.size();
}

int Model::nnrms(){
    return (int)nrms_.size();
}

Vec3f Model::vert(int i) {
    return verts_[i];
}

Vec2f Model::uvs(int i) {
    return uvs_[i];
}

Vec3f Model::nrms(int i)
{
    return nrms_[i];
}

Vec3f Model::face_vet(int facei, int idx)
{
    return verts_[faces_vet[facei * 3 + idx]];
}

Vec2f Model::face_uvs(int facei, int idx)
{
    return uvs_[faces_uvs[facei * 3 + idx]];
}

Vec3f Model::face_nrm(int facei, int idx)
{
    return nrms_[faces_nrm[facei * 3 + idx]];
}
```

顺便也贴一下我结合texture之后的triangle函数吧! 主要是实现了对纹理坐标的插值.

此处的插值方法是直接使用屏幕坐标中的重心坐标对颜色插值, 可能效果会比在纹理坐标中直接插值差一些, 但是因为没给映射函数, 我也不知道怎么在纹理中插值,.

```cpp
void new_triangle(Vec3f *pts, Vec2f *texture_pts, float *zBuffer, TGAImage &image,TGAImage& texture, float intensity)
{
    Vec2i bboxmin = Vec2i(image.get_width() - 1, image.get_height() - 1);
    Vec2i bboxmax = Vec2i(0, 0);
    for (int i = 0; i < 3; i++)
    {
        if(pts[i].x>bboxmax.x)
            bboxmax.x = pts[i].x;
        if(pts[i].y>bboxmax.y)
            bboxmax.y = pts[i].y;
        if(pts[i].x<bboxmin.x)
            bboxmin.x = pts[i].x;
        if(pts[i].y<bboxmin.y)
            bboxmin.y = pts[i].y;
    }
    for (int x = bboxmin.x; x <= bboxmax.x; x++)
    {
        for (int y = bboxmin.y; y <= bboxmax.y; y++)
        {
            Vec3f coor = barycentric(pts, Vec2i(x, y));
            if (coor.x >= 0 && coor.y >= 0 && coor.z >= 0)
            {
                float tx = coor.x * texture_pts[0].x + coor.y * texture_pts[1].x + coor.z * texture_pts[2].x;
                float ty = coor.x * texture_pts[0].y + coor.y * texture_pts[1].y + coor.z * texture_pts[2].y;
                float z = (coor.x * pts[0].z + coor.y * pts[1].z + coor.z * pts[2].z);
                //printf("tex_coor x %d y %d\n", (int)tx, (int)ty);
                //printf("color r %d g %d b %d\n", texture.get(tx, ty).r, texture.get(tx, ty).g, texture.get(tx, ty).b);
                TGAColor color = texture.get(tx, ty);
                if (zBuffer[x + y * width] < z)
                {
                    zBuffer[x + y * width] = z;
                    image.set(x, y, TGAColor(intensity * color.r, intensity * color.g, intensity * color.b, color.a));
                    // image.set(x, y, color);
                }
            }
        }
    }
}
```

最后附上我的图! 和老师的图效果稍微差一点,但不是很明显哈哈

![](https://s3.bmp.ovh/imgs/2022/05/14/400638c6ba10245b.png)
