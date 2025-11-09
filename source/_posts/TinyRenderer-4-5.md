---
categories: [技术,图形学,TinyRenderer]
tags: [图形学,C++,TinyRenderer,软件光栅化渲染器]
title: TinyRenderer 4&5
date: 2022-05-16 10:04:43
excerpt: TinyRendererWIki 4&5
---
# Lesson 4: Perspective projection

## 前言

此前我们讨论如何将三维模型内的点投影到屏幕上时,只是仅仅进行了等比例的缩放,而并没有考虑到Z轴对于缩放比例的影响!这显然是不够合理的!(因为没有考虑到透视关系的影响)(因为我们的眼睛(或者是camera)只是一个点.)

投影变换我们将借助矩阵来完成.(因为矩阵具有很好而且很方便的计算性质).

基本的线性变换有scaling shearing rotation, 矩阵的乘法不具有交换律, 不能随意改变顺序.

除了线性变换,我们还有**仿射**(affine)变换. 在线性变换中,原点的位置始终不会发生改变,而仿射变换则不然.

仿射变换的矩阵计算需要借助齐次坐标(homogeneous coordinates)

这些数学知识在此一笔带过,不想多提,具体可以看[原文](https://github.com/ssloy/tinyrenderer/wiki/Lesson-4:-Perspective-projection).

## 代码部分

我认为这部分才是重点哇,但是居然没怎么提及!

想要实现这个功能, 我们还是要做一些工作的!

工欲善其事必先利其器, 首先, 我们要先在 `geometry.h`写一个矩阵模板!

该矩阵应当是可拓展的(至少也支持4维)

应该封装好求逆和转置的函数

应该支持矩阵乘法

贴一下代码吧:

```cpp
//geometry.h内新增的Matrix类, 用于储存矩阵
class Matrix
{
	std::vector<std::vector<float>> m;
    int rows, cols;
public:
	inline int nrows();
	inline int ncols();
	static Matrix identity(int deminsion);
	std::vector<float>& operator[](const int i);
    Matrix operator*(const Matrix& a);
    Matrix transpose();
    Matrix inverse();

    friend std::ostream& operator<<(std::ostream& s, Matrix& m);
};
```

```cpp
//添加geometry.cpp文件,实现内部方法
#include <vector>
#include <cassert>
#include <cmath>
#include <iostream>
#include "geometry.h"

Matrix::Matrix(int r, int c) : m(r, std::vector<float>(c, 0.f)), rows(r), cols(c){};
//c++语法 构造初始化列表

int Matrix::nrows()
{
    return rows;
}

int Matrix::ncols()
{
    return cols;
}

Matrix Matrix::identity(int dimension)
{
    Matrix M(dimension, dimension);
    for (int i = 0; i < dimension;i++)
    {
        for (int j = 0; j < dimension;j++)
        {
            M[i][j] = (i == j) ? 1.f : 0.f;
        }
    }
}

std::vector<float>& Matrix::operator[](const int i) {
    assert(i>=0 && i<rows);
    return m[i];
}

Matrix Matrix::operator*(const Matrix& a)
{
    assert(cols == a.rows);
    Matrix res(rows, a.cols);
    for (int i=0; i<rows; i++) 
    {
        for (int j=0; j<a.cols; j++) 
        {
            for (int k=0; k<cols; k++) 
            {
                res.m[i][j] += m[i][k]*a.m[k][j];
            }
        }
    }
    return res;
}

Matrix Matrix::transpose()
{
    Matrix res(rows, cols);
    for (int i = 0; i < rows; i++)
    {
        for (int j = 0; j < cols; j++)
        {
            res[i][j] = m[j][i];
        }
    }
}

//计算逆矩阵需要一些数学功底,这里我是复制黏贴的,等我有时间再改成我自己的!
Matrix Matrix::inverse() {
    assert(rows==cols);
    // augmenting the square matrix with the identity matrix of the same dimensions a => [ai]
    Matrix result(rows, cols*2);
    for(int i=0; i<rows; i++)
        for(int j=0; j<cols; j++)
            result[i][j] = m[i][j];
    for(int i=0; i<rows; i++)
        result[i][i+cols] = 1;
    // first pass
    for (int i=0; i<rows-1; i++) {
        // normalize the first row
        for(int j=result.cols-1; j>=0; j--)
            result[i][j] /= result[i][i];
        for (int k=i+1; k<rows; k++) {
            float coeff = result[k][i];
            for (int j=0; j<result.cols; j++) {
                result[k][j] -= result[i][j]*coeff;
            }
        }
    }
    // normalize the last row
    for(int j=result.cols-1; j>=rows-1; j--)
        result[rows-1][j] /= result[rows-1][rows-1];
    // second pass
    for (int i=rows-1; i>0; i--) {
        for (int k=i-1; k>=0; k--) {
            float coeff = result[k][i];
            for (int j=0; j<result.cols; j++) {
                result[k][j] -= result[i][j]*coeff;
            }
        }
    }
    // cut the identity matrix back
    Matrix truncate(rows, cols);
    for(int i=0; i<rows; i++)
        for(int j=0; j<cols; j++)
            truncate[i][j] = result[i][j+cols];
    return truncate;
}

//这是一种给矩阵赋值的方式! 老师爱用, 我就不是很喜欢...
std::ostream& operator<<(std::ostream& s, Matrix& m) {
    for (int i=0; i<m.nrows(); i++)  
    {
        for (int j=0; j<m.ncols(); j++) 
        {
            s << m[i][j];
            if (j<m.ncols()-1) s << "\t";
        }
        s << "\n";
    }
    return s;
}
```

OK, 现在我们已经拥有矩阵了! 那么, 就简单实现一下透视效果吧!

首先我们要深刻理解不同坐标系之间的转换。

首先，在obj模型中， 物体的坐标为本地坐标， x，y范围为[1,-1]， z为[-1,0]。

接着， 物体一定是位于场景中的某个位置的， 这个位置我们称为世界坐标。（在本课中是同一个东西，还没考虑）

场景中除了物体，一定还有我们的摄像机， 如果我们认为摄像机是原点，视线是z轴， 那么物体在此坐标系下的坐标即为摄像机坐标。（下次课可能要讨论的东西）

此时， 因为我们认为摄像机是一个点， 那么就会产生透视以及透视带来的畸变效果，我们将物体依据光的直线传播进行一个透视投影， 此时我们已经成功化3D为2D。

最后， 透视完的物体并没有根据屏幕大小进行拉伸， 此时仍是一个很小的范围， 因此我们应该根据屏幕大小进行缩放。

![](https://s3.bmp.ovh/imgs/2022/05/18/d8a4450b2f3e62d9.png)

lesson4中, 我们不考虑移动摄像机的位置, 就简单的摆在(0,0,3)，因此省去了很多麻烦。

具体需要修改的的就是main函数中将模型的世界坐标转换为屏幕像素坐标那一步, 我们应该添加一个透视效果!

（纹理和光照会因为坐标变换受到影响吗？）

```cpp
//main函数
for (int j = 0; j < 3; j++)
        {
            texture_coords[j] = model->face_uvs(i, j);//老师是确认了intensity>0才去获取纹理，这样确实提高了效率！
            world_coords[j] = model->face_vet(i, j);
            //模型坐标不能简单地转换为屏幕坐标,应该经过投影!
            screen_coords[j] = trans(world_coords[j], camera, width, height);
            screen_texture_coords[j] = Vec2f((texture_coords[j].x * image.get_width()), (texture_coords[j].y * image.get_height()));
        }
```

```cpp
//trans函数以及另外两个变换函数
Vec3f trans(Vec3f world_coor, Vec3f camera, int width, int height)
{
    Matrix worldCoor = Matrix(4, 1);
    worldCoor[0][0] = world_coor.x;
    worldCoor[1][0] = world_coor.y;
    worldCoor[2][0] = world_coor.z;
    worldCoor[3][0] = 1.f;
    //puts("v2m");
    Matrix temp = project(worldCoor, camera);
    //puts("transing");
    Matrix res = viewport(temp, width/8, height/8, width*3/4, height*3/4);
    //puts("trans done");
    //将res归一并转换为Vec3f
    Vec3f screenCoor = Vec3f(res[0][0] / res[3][0], res[1][0] / res[3][0], res[2][0] / res[3][0]);
    return screenCoor;
}
Matrix project(Matrix m, Vec3f camera)
{
    Matrix projection = Matrix::identity(4);
    //puts("identity done");
    //printf("%f\n", tmp);
    projection[3][2] = -1.f / camera.z;
    //puts("project done");
    return projection * m;
}
Matrix viewport(Matrix M, int x, int y, int w, int h) {
    Matrix m = Matrix::identity(4);
    //将中心点移动到1/2width 和 1/2height的屏幕中央
    m[0][3] = x+w/2.f;
    m[1][3] = y+h/2.f;
    m[2][3] = depth/2.f;
    //将画幅拉伸到3/4屏幕那么大，留白一些
    //这里是可以自己调的， 老师就是3/4， 我认为这个大小是合适的
    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    m[2][2] = depth/2.f;
    return m*M;
}
```

OK， 工作到这里就做完啦， 欣赏自己的美图吧！

![](https://s3.bmp.ovh/imgs/2022/05/18/8b4f4440a6159741.png)

# Lesson 5: Moving the camera

## 前言

终于要结束无聊的几何部分了！

这节课更像是对上节课啥都没讲的补偿，介绍了许多基本概念！

感谢！

作者在这里要求我们使用模型中自带的法线来实现光照模型！所以我就先做了！记录一下

```cpp
//main
for (int i = 0; i < model->nfaces(); i++)
    {
        Vec3f screen_coords[3];
        Vec3f world_coords[3];
        float intensity[3];
        Vec2f texture_coords[3];
        Vec2f screen_texture_coords[3];
        for (int j = 0; j < 3; j++)
        {
            texture_coords[j] = model->face_uvs(i, j);
            world_coords[j] = model->face_vet(i, j);
            intensity[j] = model->face_nrm(i, j) * light_dir;
            //模型坐标不能简单地转换为屏幕坐标,应该经过投影!
            screen_coords[j] = trans(world_coords[j], camera, width, height);
            screen_texture_coords[j] = Vec2f((texture_coords[j].x * image.get_width()), (texture_coords[j].y * image.get_height()));
        }
        //完成插值算法
        //完成纹理映射
        Vec3f pts[3] = {screen_coords[0], screen_coords[1], screen_coords[2]};
        Vec2f texture_pts[3] = {screen_texture_coords[0], screen_texture_coords[1], screen_texture_coords[2]};
        float intensity_pts[3] = {intensity[0], intensity[1], intensity[2]};
            new_triangle(pts, texture_pts, intensity_pts, frame, image);
    }
```

```cpp
//triangle
for (int x = bboxmin.x; x <= bboxmax.x; x++)
    {
        for (int y = bboxmin.y; y <= bboxmax.y; y++)
        {
            Vec3f coor = barycentric(pts, Vec2i(x, y));
            if (coor.x >= 0 && coor.y >= 0 && coor.z >= 0)
            {
                float tx = coor.x * texture_pts[0].x + coor.y * texture_pts[1].x + coor.z * texture_pts[2].x;
                float ty = coor.x * texture_pts[0].y + coor.y * texture_pts[1].y + coor.z * texture_pts[2].y;
                float intensity = intensity_pts[0] * coor.x + intensity_pts[1] * coor.y + intensity_pts[2] * coor.z;//光照插值！
                float z = (coor.x * pts[0].z + coor.y * pts[1].z + coor.z * pts[2].z);
                TGAColor color = texture.get(tx, ty);
                if (zBuffer[x + y * width] < (int)z)
                {
                    //printf("z:%d zbuffer: %d \n", (int)z, zBuffer[x + y * width]);
                        zBuffer[x + y * width] = (int)z;
                        if(intensity<0)
                            intensity = 0;//这里是个大坑！！如果intensity<0,因为没有环境光，因此应该是全黑，就是0，我调了好久才发现！
                        image.set(x, y, TGAColor(intensity * color.r, intensity * color.g, intensity * color.b, color.a));
                        //image.set(x, y, color);
                }
            }
        }
    }
```

![](https://s3.bmp.ovh/imgs/2022/05/18/a576ddf33dc3ecb1.png)

此时（光源为 `Vec3f light_dir = Vec3f(1,-1,1).normalize();`）的效果图！（不知道为啥有点磨损，在我屏幕上效果还要更好点。。）

## 变换基坐标

正如我前面提到的一样，我不是很想讲数学， 感兴趣可以看原文，大致交待了几点：

1. 如何确定摄像机的方位： e（eye）和u（up）两个向量
2. 物体处于世界坐标，而我们需要得到物体以摄像机为原点的坐标
3. 如何确定摄像机的坐标系：c（物体）e向量直接为z'轴，x’轴为u x z' ，y’轴为z' x  x'
4. 如何变换矩阵是什么？

```cpp
void lookat(Vec3f eye, Vec3f center, Vec3f up) {
    Vec3f z = (eye-center).normalize();
    Vec3f x = cross(up,z).normalize();
    Vec3f y = cross(z,x).normalize();
    Matrix Minv = Matrix::identity();
    Matrix Tr   = Matrix::identity();
    for (int i=0; i<3; i++) {
        Minv[0][i] = x[i];
        Minv[1][i] = y[i];
        Minv[2][i] = z[i];
        Tr[i][3] = -eye[i];
    }
    ModelView = Minv*Tr;
}
```

变换矩阵为 `ModelView`， 名字来源于OPENGL

## 视口变换

视口变换将模型从[-1,1] * [-1,1] * [-1,1]的小正方体转换为 [x,x+w] * [y,y+h] * [0,d]的适于屏幕的立方体。depth取255是为了方便调试。

上节课我们都写过了。

```cpp

Matrix viewport(int x, int y, int w, int h) {
    Matrix m = Matrix::identity(4);
    m[0][3] = x+w/2.f;
    m[1][3] = y+h/2.f;
    m[2][3] = depth/2.f;

    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    m[2][2] = depth/2.f;
    return m;
}
```

---

{% blockquote %}

这是部分原文，我认为很好，值得抄录下来

{% endblockquote %}

So, let us sum up. Our models (characters, for example) are created in their own local frame ( **object coordinates** ). They are inserted into a scene expressed in  **world coordinates** . The transformation from one to another is made with matrix  **Model** . Then, we want to express it in the camera frame ( **eye coordinates** ), the transformation is called  **View** . Then, we deform the scene to create a perspective deformation with **Projection** matrix ([lesson 4](https://github.com/ssloy/tinyrenderer/wiki/Lesson-4:-perspective-projection)), this matrix transforms the scene to so-called  **clip coordinates** . Finally, we draw the scene, and the matrix transforming clip coordinates to the **screen coordinates** is called  **Viewport** .

Again, if we read a point **v** from the .obj file, then to draw it on the screen it undergoes the following chain of transformations:

```c++
Viewport * Projection * View * Model * v.
```

If you look to [this](https://github.com/ssloy/tinyrenderer/blob/10723326bb631d081948e5346d2a64a0dd738557/main.cpp) commit, then you will see the following lines:

```c++
Vec3f v = model->vert(face[j]);
screen_coords[j] =  Vec3f(ViewPort*Projection*ModelView*Matrix(v));
```

As i draw a single object only, the matrix Model is equal to identity, and i merged it with the matrix View.

解释一下最后一句话， 因为我们只渲染这一个人头，因此不需要将模型从本地坐标转换到世界坐标，因此略去。

---

## 关于法线

当模型被scaling的时候， 如果不是uniform的，那么法线的坐标不应该和坐标点一起变换！

不信你自己试试！

事实上，当物体进行坐标变换的时候，法线应当进行该坐标变换的逆变换——这是由一个简单的数学原理支持的：

![](https://s3.bmp.ovh/imgs/2022/08/31/e822383a088a2862.png)

![](https://s3.bmp.ovh/imgs/2022/08/31/f7843086a4aa9768.png)

可以看到，当坐标变换M应用于坐标上时，法线相应的应该右乘上M的逆矩阵。

经过修饰，最终写成这样的形式：

![](https://s3.bmp.ovh/imgs/2022/08/31/a86ad4081971d68a.png)

## 最终效果

![](https://s3.bmp.ovh/imgs/2022/05/27/4e41cf9dea3242cf.png)

这是我们移动摄像机后的样子，可以看到跟随着我们移动摄像机，视角也发生了变化！

移动摄像机主要是发生在Modelview矩阵，贴代码：

```cpp
Matrix modelview(Vec3f eye, Vec3f center, Vec3f up) {
    Vec3f z = (eye-center).normalize();
    Vec3f x = (up^z).normalize();
    Vec3f y = (z^x).normalize();
    Matrix res = Matrix::identity(4);
    for (int i=0; i<3; i++) {
        res[0][i] = x[i];
        res[1][i] = y[i];
        res[2][i] = z[i];
        res[i][3] = -center[i];
    }
    return res;
}
```
