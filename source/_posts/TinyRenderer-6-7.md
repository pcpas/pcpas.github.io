---
categories: [技术,图形学,TinyRenderer]
tags: [图形学,C++,TinyRenderer,软件光栅化渲染器]
title: TinyRenderer 6&7
date: 2022-08-31 16:16:16
excerpt: TinyRendererWIki 6&7
---

# Lesson 6: Shaders for the software renderer

## 前言

L6很重要。不仅是因为我们结束了数学部分进入了渲染，也是因为在此处我们对代码进行了分析和重构。

## Refactoring the source code

### 碎碎念

作者在这里对代码进行了一个分析和总结，并表示：代码写的**太多太乱**了！

很巧，笔者也是这么觉得的，甚至，笔者的代码量接近原作者的两倍！！！！！好伤心！！！！

既然如此，就跟着作者的指引来重构一下自己小小渲染器的内容吧~我们应该只把有关渲染的一些**可调整**的参数和某些特定、需要修改的任务放在主函数中，而那些只是提供一个接口的渲染、计算函数不需要进一步修改，就将其放在别的特定的文件内就可以了。

想要做到这一点，就必须深刻理解整个渲染的过程。所以，跟着笔者的脚步对整个渲染进行一个梳理吧！

### 前置文件

首先，想要完成渲染需要三个头文件的支持：

```
Geometry.h
//这个头文件提供了一系列的数学计算函数，为向量和矩阵的计算提供了巨大的便利
tgaimage.h
//这个头文件允许我们读写tga图像
model.h
//这个头文件给了我们一个读取模型的途径
```

这三个头文件不需要我们倾注太多心思，因为是基础性的，但是也可以花时间研究一下。特别是`model.h` ，因为只有深刻理解我们如何储存模型的数据，才能做到合理的取用这些数据，这也是前几节课的一个难点。

首先我们要理解一个需要被渲染的模型是怎么构成的：

1. 模型 -----> 无数的三角形面 -----> 每个面由三个顶点组成 ：顶点坐标

2. 模型每个面（或点）的法线

3. 模型需要贴图（纹理），无数个纹理坐标（与顶点坐标一一对应）
4. 模型是三维的，贴图是二维的，因此才需要一一对应关系进行转换：对应表

在一个模型文件中（.obj），包含以下信息：

（初始模型的坐标均在一个[-1,1]^3的立方体内）

```
//所有顶点的坐标（在物体自身的坐标系中）
v -0.171097 0.299996 0.415616
……
# 1258 vertices

//所有顶点对应的点在纹理贴图上的坐标（数量不相等，why？）
vt  0.532 0.923 0.000
……
# 1339 texture vertices

//所有法线向量（看来是每个顶点的法线）
vn  0.001 0.482 -0.876
# 1258 vertex normals

//每个面的信息
f 1201/1249/1201 1202/1248/1202 1200/1246/1200
//格式： f （顶点索引/uv点索引/法线索引）* 3 （表示组成这个face的3个顶点）
# 2492 faces
```

通过读取这个文件，我们自然能得到模型的一切信息。

欧克，这三个头文件相当于为我们清除了一切阻碍，我们需要做的就是聚精会神的剖析渲染的过程！

### 渲染过程

这个过程我们用文字夹杂着代码的形式来说，同时因为是重新整理，所以顺序上没有那么循循善诱。

首先，我们要设定整个场景的参数：

```c++
//设定一些宏
const TGAColor white = TGAColor(255, 255, 255, 255);
const TGAColor red = TGAColor(255, 0, 0, 255);
const TGAColor green = TGAColor(0, 255, 255, 0);
//初始化模型
Model *model = NULL;
//设定画布大小
const int width = 800;
const int height = 800;
const int depth = 255;
//设定光源、照相机、坐标系
Vec3f light_dir = Vec3f(1, -1, 1).normalize();
Vec3f camera(1.f, 1.f, 3.f);
Vec3f center(0.f, 0.f, 0.f);
Vec3f up(0.f, 1.f, 0.f);
```

{% note primary%}

我想先强调一个渲染的最基本的概念：你所做的一切，都是为了得到屏幕上每个像素点的颜色。

我们做了一大堆一大堆的变化和计算，为了什么呢？没错，就是为了确定物体的每个点的屏幕坐标，这样屏幕上的像素点就可以根据周围这些点的纹理颜色设定自己的颜色了。

前面提过这个变换顺序：

```
Viewport * Projection * View * Model * vertex
```

Model变换是为了将每个点的坐标从物体坐标转换到世界坐标；

View变换是为了将每个点的坐标从世界坐标转换到摄像机（眼睛）坐标系中；

Projection变换是为了将每个点透视投影到屏幕上。这里可能比较抽象：

我们认为眼睛是一个点，那么我们看到物体是满足近大远小的透视规律的，因为屏幕只是一个平面，因此物体在屏幕上的x,y坐标必然会受到深度的影响。投影就是为了算出每个点的屏幕坐标。

ViewPort变换是为了拉伸画面。我们知道，这些变换一直是在[-1,1]^3的立方体的前提下进行的，要把画面放大到指定的高度和宽度。至于深度，因为我们只需要知道相对关系，其实绝对数值意义不大，一般为255.

这样，就可以确定模型每个点的屏幕坐标了。屏幕的每个像素点依据附近的模型面（或点）的颜色计算出显示什么颜色，就ok了。

{% endnote %}

下面咱们来一起看渲染的过程：

（蓝色的是我们无法改变的，而橘色的是我们可以随意改造的）

![](https://s3.bmp.ovh/imgs/2022/08/31/88113a7c13ebb342.png)

我们现在已经获取到了顶点的数据，通过预处理后，使用Vertex Shader对其进行变换，使模型中每个点的坐标转换为屏幕坐标，并算出相应的光照。

接着fragment Shader将会读取其光照和原纹理，计算出像素点的实际颜色，并进行上色。

不对一些细枝末节的过程进行详细说明，我们就得到了勉强可看的渲染结果啦！

所以可以看出，过程其实是：

模型 ——> 特定的shader进行渲染 ——> 结果图

所以将shader抽离出来是很有必要的！

因此，建立shader模板类：

```c++
//创建一个shader模板类
class IShader
{
public:
    virtual ~IShader();
    virtual Vec4f vertex(int iface, int nthvert) = 0;
    virtual bool fragment(Vec3f bar, TGAColor &color) = 0;
};
```

同时，因为投影变换矩阵和画三角形的函数相对固定，我们也将其抽离出来：

```c++
void viewport(int x, int y, int w, int h)
{
    Viewport = Matrix::identity();
    //将中心点移动到1/2width 和 1/2height的屏幕中央
    Viewport[0][3] = x + w / 2.f;
    Viewport[1][3] = y + h / 2.f;
    //深度
    Viewport[2][3] = 255.f / 2.f;
    //将画幅拉伸到3/4屏幕那么大，留白一些
    //这里是可以自己调的， 老师就是3/4， 我认为这个大小是合适的
    Viewport[0][0] = w / 2.f;
    Viewport[1][1] = h / 2.f;
    Viewport[2][2] = 255.f / 2.f;
}

void projection(float coeff)
{
    Projection = Matrix::identity();
    Projection[3][2] = coeff;
}

void lookat(Vec3f eye, Vec3f center, Vec3f up)
{
    Vec3f z = (eye - center).normalize();
    Vec3f x = cross(up, z).normalize();
    Vec3f y = cross(z, x).normalize();
    ModelView = Matrix::identity();
    for (int i = 0; i < 3; i++)
    {
        ModelView[0][i] = x[i];
        ModelView[1][i] = y[i];
        ModelView[2][i] = z[i];
        ModelView[i][3] = -center[i];
    }
}

Vec3f barycentric(Vec2f A, Vec2f B, Vec2f C, Vec2f P)
{
    Vec3f s[2];
    for (int i = 2; i--;)
    {
        s[i][0] = C[i] - A[i];
        s[i][1] = B[i] - A[i];
        s[i][2] = A[i] - P[i];
    }
    Vec3f u = cross(s[0], s[1]);
    if (std::abs(u[2]) > 1e-2) // dont forget that u[2] is integer. If it is zero then triangle ABC is degenerate
        return Vec3f(1.f - (u.x + u.y) / u.z, u.y / u.z, u.x / u.z);
    return Vec3f(-1, 1, 1); // in this case generate negative coordinates, it will be thrown away by the rasterizator
}

void triangle(Vec4f *pts, IShader &shader, TGAImage &image, TGAImage &zbuffer)
{
    Vec2f bboxmin(std::numeric_limits<float>::max(), std::numeric_limits<float>::max());
    Vec2f bboxmax(-std::numeric_limits<float>::max(), -std::numeric_limits<float>::max());
    for (int i = 0; i < 3; i++)
    {
        for (int j = 0; j < 2; j++)
        {
            bboxmin[j] = std::min(bboxmin[j], pts[i][j] / pts[i][3]);
            bboxmax[j] = std::max(bboxmax[j], pts[i][j] / pts[i][3]);
        }
    }
    Vec2i P;
    TGAColor color;
    for (P.x = bboxmin.x; P.x <= bboxmax.x; P.x++)
    {
        for (P.y = bboxmin.y; P.y <= bboxmax.y; P.y++)
        {
            Vec3f c = barycentric(proj<2>(pts[0] / pts[0][3]), proj<2>(pts[1] / pts[1][3]), proj<2>(pts[2] / pts[2][3]), proj<2>(P));
            float z = pts[0][2] * c.x + pts[1][2] * c.y + pts[2][2] * c.z;
            float w = pts[0][3] * c.x + pts[1][3] * c.y + pts[2][3] * c.z;
            int frag_depth = std::max(0, std::min(255, int(z / w + .5)));
            if (c.x < 0 || c.y < 0 || c.z < 0 || zbuffer.get(P.x, P.y)[0] > frag_depth)
                continue;
            bool discard = shader.fragment(c, color);
            if (!discard)//为什么要弄一个bool？——方便丢弃像素（在之后的课程中会出现的）
            {
                zbuffer.set(P.x, P.y, TGAColor(frag_depth));
                image.set(P.x, P.y, color);
            }
        }
    }
}
```

好了，此时我们的主函数是这样的：

（笔者这里非常偷懒的全部使用的老师的模板：因为笔者的几个头文件一开始是自己写的，在扩充的过程中成为屎山，且与主函数的协调都存在巨大问题。。。完全重构耗时太长，因此只能从这里先借鉴一下范本，有能力还是自己写，有助于深入了解。笔者已经上过GAMES101，很轻易能看懂，因此偷个小懒啦~）

```c++
//高氏shader——最初级的shader
struct GouraudShader : public IShader {
    Vec3f varying_intensity; // written by vertex shader, read by fragment shader

    virtual Vec4f vertex(int iface, int nthvert) {
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        gl_Vertex = Viewport*Projection*ModelView*gl_Vertex;     // transform it to screen coordinates
        varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir); // get diffuse lighting intensity
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        float intensity = varying_intensity*bar;   // interpolate intensity for the current pixel
        color = TGAColor(255, 255, 255)*intensity; // well duh
        return false;                              // no, we do not discard this pixel
    }
};

int main(int argc, char** argv) {
    //获取模型
    if (2==argc) {
        model = new Model(argv[1]);
    } else {
        model = new Model("obj/african_head.obj");
    }
    //初始化变换矩阵
    lookat(eye, center, up);
    viewport(width/8, height/8, width*3/4, height*3/4);
    projection(-1.f/(eye-center).norm());
    //初始化光线
    light_dir.normalize();
    //初始纹理和深度图
    TGAImage image  (width, height, TGAImage::RGB);
    TGAImage zbuffer(width, height, TGAImage::GRAYSCALE);
    //获取高氏Shader
    GouraudShader shader;
    //借助shader渲染模型
    for (int i=0; i<model->nfaces(); i++) {
        Vec4f screen_coords[3];
        for (int j=0; j<3; j++) {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, image, zbuffer);
    }
    //输出
    image.  flip_vertically(); // to place the origin in the bottom left corner of the image
    zbuffer.flip_vertically();
    image.  write_tga_file("output.tga");
    zbuffer.write_tga_file("zbuffer.tga");

    delete model;
    return 0;
}
```

可以看到，我们这里使用的是高氏Shader，此后，我们会学习更多的Shader，同时我们自己也应该学会自己写Shader，比如我们马上就要迎来的下一课。

## 修改Shader的第一次尝试

尝试一下通过修改Shader来产生各种不同的渲染效果吧！

这是作者的示例：可以看到，作者只允许亮度有6个不同的值，而且初始颜色被设定为了橘黄色，可以想象这样会是一种厚涂的风格。

```c++
  virtual bool fragment(Vec3f bar, TGAColor &color) 
  {
        float intensity = varying_intensity*bar;
        if (intensity>.85) intensity = 1;
        else if (intensity>.60) intensity = .80;
        else if (intensity>.45) intensity = .60;
        else if (intensity>.30) intensity = .45;
        else if (intensity>.15) intensity = .30;
        else intensity = 0;
        color = TGAColor(255, 155, 0)*intensity;
        return false;
    }
```

![](https://s3.bmp.ovh/imgs/2022/09/03/a51d60c7d28aa665.png)

## Normalmapping

textrue不仅可以储存RGB颜色，也可以存储别的，非常丰富的信息，比如法线。

<img src="https://s3.bmp.ovh/imgs/2022/09/03/24d8531246467622.png" style="zoom:50%;" />

这就是一张典型的法线贴图，存放了每个点的法线信息。

既然说到法线，那我们就来扯一扯法线的问题。还记得之前我们说到的如果模型进行了坐标变换，那么法线也需要进行坐标变换吗?那为什么我们一直没有提到关于发现的坐标变换呢？一起来看看吧。

首先是一开始的高氏Shader：

```c++
struct GouraudShader : public IShader
{
    Vec3f varying_intensity; // written by vertex shader, read by fragment shader

    virtual Vec4f vertex(int iface, int nthvert)
    {
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert));                               // read the vertex from .obj file
        gl_Vertex = Viewport * Projection * ModelView * gl_Vertex;                             // transform it to screen coordinates
        varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert) * light_dir); // get diffuse lighting intensity
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color)
    {
        float intensity = varying_intensity * bar;   // interpolate intensity for the current pixel
        color = TGAColor(255, 255, 255) * intensity; // well duh
        return false;                                // no, we do not discard this pixel
    }
};
//我们需要使用到法线的地方在哪里？—— 计算光照
//varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert) * light_dir); 
//可以看到，我们根本没管这里点经过投影之后的法线变换，也没管光线在拉伸后的空间后的变化，因此效果确实会差一些！
//此处light_dir是否需要拉伸还是看你给的是什么坐标系下的light_dir.
```

我们修改一下：

```c++
struct Shader : public IShader
{
    mat<2, 3, float> varying_uv;  // same as above
    mat<4, 4, float> uniform_M;   //  Projection*ModelView
    mat<4, 4, float> uniform_MIT; // (Projection*ModelView).invert_transpose()

    virtual Vec4f vertex(int iface, int nthvert)
    {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport * Projection * ModelView * gl_Vertex;    // transform it to screen coordinates
    }

    virtual bool fragment(Vec3f bar, TGAColor &color)
    {
        Vec2f uv = varying_uv * bar; // interpolate uv for the current pixel
        Vec3f n = proj<3>(uniform_MIT * embed<4>(model->normal(uv))).normalize();
        Vec3f l = proj<3>(uniform_M * embed<4>(light_dir)).normalize();
        float intensity = std::max(0.f, n * l);
        color = model->diffuse(uv) * intensity; // well duh
        return false;                           // no, we do not discard this pixel
    }
};

//这里我们是怎么计算光照的呢？
//因为提供了法线贴图，所以不需要点对点的单独提取法线了，只要直接插值算出每个三角形对应的uv，颜色和法线都可以通过uv确定。取得三角形的法线后再直接和光线点乘得到系数。
```

最后我们对比一下这两种方法的效果差距：

<img src="https://s3.bmp.ovh/imgs/2022/09/03/879521a02efd38ca.png" style="zoom: 33%;" />

<img src="https://s3.bmp.ovh/imgs/2022/09/03/77e0fde064a19dc0.png" style="zoom:33%;" />

## Specular mapping

我们看着这个人像，仍然不满意于渲染的结果。为什么呢？

光照还是不够真实！众所周知，光在打到物体时会发生反射，分为镜面反射和漫反射。而且我们是如何处理光照的？——非常直觉性的”法线和光线方向一致则亮，否则则暗”，这是一种模拟漫反射强度的有效方法，但是Phong提出了一个更好的光照模型，我们一起来学习一下：

![](https://s3.bmp.ovh/imgs/2022/09/03/34a77862e0fac16d.png)

<img src="https://s3.bmp.ovh/imgs/2022/09/03/5d0696b25eb6b6d6.png" style="zoom:50%;" />

下面给出代码：

```c++
struct Shader : public IShader {
    mat<2,3,float> varying_uv;  // same as above
    mat<4,4,float> uniform_M;   //  Projection*ModelView
    mat<4,4,float> uniform_MIT; // (Projection*ModelView).invert_transpose()

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec2f uv = varying_uv*bar;
        Vec3f n = proj<3>(uniform_MIT*embed<4>(model->normal(uv))).normalize();
        Vec3f l = proj<3>(uniform_M  *embed<4>(light_dir        )).normalize();
        Vec3f r = (n*(n*l*2.f) - l).normalize();   // reflected light
        float spec = pow(std::max(r.z, 0.0f), model->specular(uv));//可以看到，又出现了一个高光贴图，用于给出每处的粗糙度，有助于让光滑的地方获得更强的反光。
        float diff = std::max(0.f, n*l);
        TGAColor c = model->diffuse(uv);
        color = c;
        for (int i=0; i<3; i++) color[i] = std::min<float>(5 + c[i]*(diff + .6*spec), 255);
        return false;
    }
};
```

这里作者写的

```c++
Vec3f r = (n*(n*l*2.f) - l).normalize();
float spec = pow(std::max(r.z, 0.0f), model->specular(uv));
```

感觉有点无厘头（

按理来说应该把r和v先求点乘，然后再power多次，得到高光才对。合理猜测是因为此时eye的位置比较取巧，所以才不需要考虑v，但是v还是很重要的。

最后出个结果吧。

![](https://s3.bmp.ovh/imgs/2022/09/03/2416957d30229da8.png)

## 总结

这节课收获还是很多的！

我们已经可以渲染的像模像样了，但是还远远达不到“好”和“真实”。

我们虽然简单模拟了光，但是没有考虑阴影呢！

# Lesson 6bis: tangent space normal mapping

## 前言

很神奇，中间插了一截来讲normalMapping，那我们在讲阴影之前就先看一看NormalMapping吧！

先链接一篇文章，因为这里涉及到了很多线性代数，数学不好的话肯定会很迷惑。

[法线贴图那些事儿](https://zhuanlan.zhihu.com/p/143321426)

我们思考一个问题：

如果我们想用这个模型做一段动画，那么该怎么获取每一帧物体的位置、纹理和法线呢？

首先是位置：每个三角面的位置由动画本身决定，不需要我们去获取。

纹理：每个三角形对应的uv贴图的坐标是一定的，在运动过程中纹理不会发生改变，因此不需要处理

法线：每个三角形的法线由三角形三个顶点的法线计算而来，而这由模型文件本身给出，因此，在三角形顶点跟随动画运动的过程中，法线无法确定了！

因此（不构成直接的因果关系，只是基于切线空间的法线贴图有这个好处），我们注意到了tangent space normal mapping！

切线空间的法线贴图存储着每个顶点的相对法向量，因此，无论顶点怎么动，只要用合适的数学将其提取出来，就一定会是正确的法向！

好了，接下来我们来学习一下吧！

## 代码展示

和往常一样，我不会展示具体的数学推导。（因为解释起来太麻烦+我也不一定正确)，所以指路[原帖](https://github.com/ssloy/tinyrenderer/wiki/Lesson-6bis:-tangent-space-normal-mapping)。

```c++
//只有Shader发生了变化，其他部分和以前一样。
struct Shader : public IShader {
    mat<2,3,float> varying_uv;  // triangle uv coordinates, written by the vertex shader, read by the fragment shader
    mat<4,3,float> varying_tri; // triangle coordinates (clip coordinates), written by VS, read by FS
    mat<3,3,float> varying_nrm; // normal per vertex to be interpolated by FS
    mat<3,3,float> ndc_tri;     // triangle in normalized device coordinates

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        varying_nrm.set_col(nthvert, proj<3>((Projection*ModelView).invert_transpose()*embed<4>(model->normal(iface, nthvert), 0.f)));
        Vec4f gl_Vertex = Projection*ModelView*embed<4>(model->vert(iface, nthvert));
        varying_tri.set_col(nthvert, gl_Vertex);
        ndc_tri.set_col(nthvert, proj<3>(gl_Vertex/gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec3f bn = (varying_nrm*bar).normalize();
        Vec2f uv = varying_uv*bar;

        mat<3,3,float> A;
        A[0] = ndc_tri.col(1) - ndc_tri.col(0);
        A[1] = ndc_tri.col(2) - ndc_tri.col(0);
        A[2] = bn;

        mat<3,3,float> AI = A.invert();

        Vec3f i = AI * Vec3f(varying_uv[0][1] - varying_uv[0][0], varying_uv[0][2] - varying_uv[0][0], 0);
        Vec3f j = AI * Vec3f(varying_uv[1][1] - varying_uv[1][0], varying_uv[1][2] - varying_uv[1][0], 0);

        mat<3,3,float> B;//这就是切线空间的基了
        B.set_col(0, i.normalize());
        B.set_col(1, j.normalize());
        B.set_col(2, bn);

        Vec3f n = (B*model->normal(uv)).normalize();

        float diff = std::max(0.f, n*light_dir);
        color = model->diffuse(uv)*diff;

        return false;
    }
};
```



# Lesson 7: Shadow mapping

## 前言

这节课我们来学习添加阴影。（阴影分为硬阴影和软阴影，此处我们添加硬阴影）

## 阴影形成原理

我们都知道，阴影的形成是因为光源发出的光线被中间的物体遮挡，因此形成了阴影。

借助这种想法，轻易的就可以理解阴影的求法：做一个类似于zBuffer的shadowBuffer，只渲染最上层的物体。

如何形成呢？我们可以将眼睛（eye）的位置放在光源，这样我们看过去储存的zBuffer其实就是要求的shadowBuffer。

```c++
struct DepthShader : public IShader
{
    mat<3, 3, float> varying_tri;

    DepthShader():varying_tri(){}

    virtual Vec4f vertex(int iface, int nthvert)
    {
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert));
        gl_Vertex = shadowM * gl_Vertex;
        varying_tri.set_col(nthvert, proj<3>(gl_Vertex / gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color)
    {
        Vec3f p = varying_tri * bar;
        color = TGAColor(255, 255, 255) * (p.z / 255);
        return false;
    }
};

//此处shadowM矩阵为： 
lookat(light_dir, center, up);//将眼睛放在光源处
viewport(width / 8, height / 8, width * 3 / 4, height * 3 / 4);
projection(-1.f / (eye - center).norm());
shadowM = Viewport * Projection * ModelView;
```

## 修改Shader

那么如何修改我们的Shader也就很清晰了：在我们PhongShader的基础上添加一个前置步骤：计算ShadowBuffer，这样在PhongShader渲染时对于有阴影的部分就可以加以处理了。

```c++
//此处展示我们编写的新的PhongShader
struct PhongShader : public IShader
{
    mat<4, 4, float> uniform_M;       //  Projection*ModelView
    mat<4, 4, float> uniform_MIT;     // (Projection*ModelView).invert_transpose()
    mat<4, 4, float> uniform_Mshadow; // transform framebuffer screen coordinates to shadowbuffer screen coordinates
    mat<2, 3, float> varying_uv;      // triangle uv coordinates, written by the vertex shader, read by the fragment shader
    mat<3, 3, float> varying_tri;     // triangle coordinates before Viewport transform, written by VS, read by FS

    PhongShader(Matrix M, Matrix MIT, Matrix MS) : uniform_M(M), uniform_MIT(MIT), uniform_Mshadow(MS), varying_uv(), varying_tri() {}

    virtual Vec4f vertex(int iface, int nthvert)
    {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = Viewport * Projection * ModelView * embed<4>(model->vert(iface, nthvert));
        varying_tri.set_col(nthvert, proj<3>(gl_Vertex / gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color)
    {
        Vec4f sb_p = uniform_Mshadow * embed<4>(varying_tri * bar); // corresponding point in the shadow buffer
        sb_p = sb_p / sb_p[3];                           
        float shadow = .3 + .7 * (shadowDepth.get((int)sb_p[0], (int)sb_p[1])[0] < (sb_p[2]+10)); // magic coeff to avoid z-fighting
        //这里这个+10需要注意一下，之后会解释，是为防止z—fighting
        //可以看到，为了模仿软阴影，即使被遮挡住了也不会全黑，而是保留0.3的系数
        Vec2f uv = varying_uv * bar;                                              // interpolate uv for the current pixel
        Vec3f n = proj<3>(uniform_MIT * embed<4>(model->normal(uv))).normalize(); // normal
        Vec3f l = proj<3>(uniform_M * embed<4>(light_dir)).normalize();           // light vector
        Vec3f r = (n * (n * l * 2.f) - l).normalize();                            // reflected light
        float spec = pow(std::max(r.z, 0.0f), model->specular(uv)); //为什么只考虑z? 这里也是一个trick,因为眼睛一定在z轴上,只需要取z分量就可以了
        float diff = std::max(0.f, n * l);
        TGAColor c = model->diffuse(uv);
        for (int i = 0; i < 3; i++)
           color[i] = std::min<float>(20 + c[i]*shadow*(1.2*diff + .6*spec), 255);
        return false;
    }
};
```

然后在主函数中进行渲染：

```c++
    // GetDepth
    lookat(light_dir, center, up);
    viewport(width / 8, height / 8, width * 3 / 4, height * 3 / 4);
    projection(-1.f / (eye - center).norm());
    shadowM = Viewport * Projection * ModelView;
    DepthShader Dshader;
    for (int i = 0; i < model->nfaces(); i++)
    {
        Vec4f screen_coords[3];
        for (int j = 0; j < 3; j++)
        {
            screen_coords[j] = Dshader.vertex(i, j);
        }
        triangle(screen_coords, Dshader, shadowImage, shadowDepth);
    }
    shadowImage.flip_vertically();
    shadowImage.write_tga_file("shadow.tga");

    // PhongShading
    lookat(eye, center, up);
    PhongShader shader(ModelView, (Projection * ModelView).invert_transpose(), shadowM * (Viewport * Projection * ModelView).invert());
    for (int i = 0; i < model->nfaces(); i++)
    {
        Vec4f screen_coords[3];
        for (int j = 0; j < 3; j++)
        {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, image, depth);
    }
    //输出
    image.flip_vertically(); // to place the origin in the bottom left corner of the image
    image.write_tga_file("output.tga");

```

总的来说原理还是很好理解的，但是这里面的数学确实让人有点伤脑筋，外加这个wiki的作者的代码经常反反复复的重构和更新，导致在复现的过程中还是遇到不少麻烦，特别是会出现一些不太理解的magic的变量和数字。但是如果能清晰的理解里面的数学，把数学的部分封装好，我觉得应该能简单很多！

最后看两张成品图吧！

<img src="https://s3.bmp.ovh/imgs/2022/09/05/c075281389264e5c.png" style="zoom:50%;" />

<img src="https://s3.bmp.ovh/imgs/2022/09/05/74ca9782bd5a04b9.png" style="zoom: 67%;" />

## 一个小问题

通过成品图（特别是第二张图）我们可以看到，模型身上有一些非常丑陋的闪烁的阴影，这是为什么呢？

这是一种特别的现象：Z-fighting。

链接一篇知乎专栏：[<渲染基础>-3D渲染中的Z-fighting现象](https://zhuanlan.zhihu.com/p/78769570)

CSDN : [Z-Fighting问题解决方案实例](https://blog.csdn.net/chenweiyu11962/article/details/113542402)

在作者的教程中，他直接添加了一个magic number：43.34来解决这个问题

这实际上是制造了一个偏移，这样可能可以整体性的去除一些闪烁？？不太懂

```c++
float shadow = .3 + .7 * (shadowDepth.get((int)sb_p[0], (int)sb_p[1])[0] < (sb_p[2]+10)); // magic coeff to avoid z-fighting
```

我使用的+10，没有什么特别大的区别，但是可以看出这种方法只能一定程度上解决这个问题，该闪烁还是会闪烁。
