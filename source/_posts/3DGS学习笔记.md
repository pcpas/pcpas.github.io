---
title: 3DGS学习笔记
date: 2024-04-21 22:21:55
excerpt: 自学3D Gaussian Splatting路上记录的笔记和问题
categories: [技术,AI]
tags: [CV,3DGS]
---
## 1 概述

3D 高斯泼溅技术（3D Gaussian Splatting，简称3DGS）由 [3D Gaussian Splatting for Real-Time Radiance Field Rendering](https://huggingface.co/papers/2308.04079) 一文首次提出。作为一种光栅化技术，3D 高斯泼溅可用于实时且逼真地渲染从一小组图像中学到的场景。

具体来说，给定一组图像，我们利用SFM方法将其转化为一组稀疏点云，以此为基元3D高斯的均值初始值，通过迭代优化，得到完整的场景模型。

## 2 3D高斯

3D高斯可以看作一个椭球体，位置由其均值决定，大小由其协方差决定；高斯的颜色使用球谐函数表示，因为从不同的方向看上去高斯可能呈现不同的颜色；最后每个3D高斯还拥有自己的不透明度。

$$
p(x)=\frac{1}{ | \Sigma|^{1 / 2} (2 \pi)^{3 / 2}} \exp \left(-\frac{1}{2}(\mathbf{x}-\mu)^{T} \Sigma^{-1}(\mathbf{x}-\mu)\right)
$$

三维高斯的协方差矩阵是一个半正定的实对称矩阵，因此它一定可以被正交对角化：

$$
\Sigma=Q \Lambda Q^T
$$

其中$\Lambda$是特征值矩阵，又因为其半正定性，因此可以进行分解：

$$
\Sigma=Q \Lambda^{\frac{1}{2}} \Lambda^{\frac{1}{2}}Q^T
$$

其中$\Lambda^{\frac{1}{2}}$是对角矩阵，其转置等于自身，因此可以写为：

$$
\Sigma=(Q\Lambda^{\frac{1}{2}})((\Lambda^{\frac{1}{2}})^TQ^T)
$$

这里的$\Lambda^{\frac{1}{2}} $和$Q$有自己的物理意义，可以理解为高斯椭球的Scaling和Rotating，因此有：

$$
\Sigma=RSS^TR^T
$$

## 3 初始化

首先，我们需要获取一组欲建模场景的图像，并使用运动恢复结构 (Structure from Motion，SfM) 方法从一组图像中估计出3D点云。我们可以直接调用 [COLMAP](https://colmap.github.io/) 库来完成这一步。

![image-20240429162200528](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240429162200528.png)

接下来，把每个点建模成一个 3D 高斯图像。从 SfM 数据中，我们能推断出每个高斯图像的位置和颜色。这对于一般的光栅化已经够用了，但如果要产生更高质量的表征的话，我们还需要对每个高斯图像进行训练，以推断出更精细的位置和颜色，并推断出协方差（即大小）和透明度。

## 4 训练

与神经网络类似，3DGS使用随机梯度下降法进行训练，但这里没有神经网络的层的概念 (都是 3D 高斯函数)。

训练步骤如下:

1. 用当前所有可微高斯图像渲染出图像 (稍后详细介绍)
2. 根据渲染图像和真实图像之间的差异计算损失
3. 根据损失调整每个高斯图像的参数
4. 根据情况对当前相关高斯图像进行自动致密化及修剪（densify_and_prune）

步骤 1-3 比较简单，下面稍微解释一下第 4 步的工作:

<img src="https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240421222005501.png" alt="image-20240421222005501" style="zoom:50%;" />

- 如果某高斯图像的梯度很大 (即它错得比较离谱)，则对其进行分割或克隆
  - 如果该高斯图像很小，则克隆它
  - 如果该高斯图像很大，则将其分割
- 如果该高斯图像的 alpha 太低，则将其删除

这么做能帮助高斯图像更好地拟合精细的细节，同时修剪掉不必要的高斯图像。

## 5 可微的高斯光栅化

渲染器可微代表着我们可以用随机梯度下降法来训练它，因此这样的渲染方法非常重要。

![image-20240421221359455](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240421221359455.png)

上面是3D GS 光栅化的示意图。

(a) Splatting步骤将 3D 高斯函数投影到2D图像空间中。

(b) 3D GS 将图像分成多个不重叠的块，即tile。

(c) 3D GS 复制覆盖多个瓷砖的高斯函数，为每个副本分配一个标识符，即tile ID。

(d) 通过渲染排序后的高斯函数，我们可以获得tile内的所有像素。

那么3D高斯函数该如何进行投影，每个像素点最终的颜色又是如何计算出来的呢？

首先我们看一个NeRf-style的体渲染方程：

$$
\begin{aligned}
C(p) & =\sum_{i=1}^{N} c_{i}\left(1-\exp \left(-\sigma_{i} \delta_{i}\right)\right) T_{i} \\
& =\sum_{i=1}^{N} c_{i}\left(1-\exp \left(-\sigma_{i} \delta_{i}\right)\right) \exp \left(-\sum_{j=1}^{i-1} \sigma_{j} \delta_{j}\right) \\
& =\sum_{i=1}^{N} c_{i}( \underbrace{1-\exp \left(-\sigma_{i} \delta_{i}\right))}_{\alpha_{i}} \prod_{j=1}^{i-1} \underbrace{\exp \left(-\sigma_{j} \delta_{j}\right)}_{1-\alpha_{j}} \\
& =\sum_{i=1}^{N} c_{i} \alpha_{i} \underbrace{\prod_{j=1}^{i-1}\left(1-\alpha_{j}\right)}_{\text {transmittance }}
\end{aligned}
$$

其中$\sigma$是体密度，$\delta$是采样间距，$\alpha$是不透明度。

3D高斯公式为：

$$
p(x)=\frac{1}{ | \Sigma|^{1 / 2} (2 \pi)^{3 / 2}} \exp \left(-\frac{1}{2}(\mathbf{x}-\mu)^{T} \Sigma^{-1}(\mathbf{x}-\mu)\right)
$$

这里我们将$p(x)$理解为这个高斯对空间中某点$x$的影响。我们忽略其归一化项，通过不透明度进行加权，得到公式：

$$
f_{i}(p)=\sigma\left(\alpha_{i}\right) \exp \left(-\frac{1}{2}\left(p-\mu_{i}\right) \Sigma_{i}^{-1}\left(p-\mu_{i}\right)\right)
$$

其中不透明度为$\sigma(\alpha_{i})$，应用 sigmoid 函数将参数映射到 [0, 1] 区间。

因此上述的体渲染方程就可以被改写为：

$$
C(p)=\sum_{i \in N} c_{i} f_{i}^{2 D}(p) \underbrace{\prod_{j=1}^{i-1}\left(1-f_{j}^{2 D}(p)\right)}_{\text {transmittance }}
$$

其中$f_i^{2D}(p)$表示第$i$个3D高斯函数的2D投影在$p$点的不透明度。

如何进行投影呢？

首先，我们需要通过相机的外参矩阵View Matrix $W$ 将坐标从世界坐标系转换到相机坐标系：

$$
W=
[R \mid \boldsymbol{t}]=\left[\begin{array}{lll|l}
r_{1,1} & r_{1,2} & r_{1,3} & t_{1} \\
r_{2,1} & r_{2,2} & r_{2,3} & t_{2} \\
r_{3,1} & r_{3,2} & r_{3,3} & t_{3}
\end{array}\right]
$$

接着，我们需要根据相机的内参矩阵计算出其投影矩阵。投影矩阵的作用是将坐标从相机坐标系转换为光线坐标系，也可以理解为图片坐标系。

相机内参矩阵的一般形式可以写为：

$$
K=\left(\begin{array}{ccc}
f_{x} & s & x_{0} \\
0 & f_{y} & y_{0} \\
0 & 0 & 1
\end{array}\right)
$$

这样的形式是根据相机本身的性质，如焦距和主点位置决定的。如果我们给出投影前视椎体的对应坐标，一般性地，我们也可以将透视矩阵写作：

$$
K=
\left(\begin{array}{cccc}
\frac{2 n}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
0 & \frac{2 n}{t-b} & \frac{t+b}{t-b} & 0 \\
0 & 0 & \frac{-(f+n)}{f-n} & \frac{-2 f n}{f-n} \\
0 & 0 & -1 & 0
\end{array}\right)
$$

值得注意的是，这里给出的透视矩阵是遵循[OpenGL规范](https://www.songho.ca/opengl/gl_projectionmatrix.html)的，矩阵的形式根据坐标系和正则立方体的不同，形式上会产生差别。

<img src="https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240429172822712.png" alt="投影矩阵如何发挥作用" style="zoom:50%;" />

这里我们根据已知数据，使用的投影矩阵为：

$$
K=\left[\begin{array}{cccc}
2 f_{x} / w & 0 & 0 & 0 \\
0 & 2 f_{y} / h & 0 & 0 \\
0 & 0 & (f+n) /(f-n) & -2 f n /(f-n) \\
0 & 0 & 1 & 0
\end{array}\right]
$$

其中$(f_x,f_y)$是相机的焦距，$(w,h)$是输出图片的尺寸，$(n,f)$是近远裁剪平面。

此时，求投影世界坐标系中的3D高斯的均值$\boldsymbol{\mu}$在NDC坐标系的坐标$\boldsymbol{t}$公式为：

$$
\boldsymbol{t}=
\begin{array}{c}
\left[\begin{array}{c}
t_x \\
t_y \\
t_z \\
t_w
\end{array}\right]=K W\left[\begin{array}{c}
\mu_{x} \\
\mu_{y} \\
\mu_{z} \\
1
\end{array}\right] \\
\end{array}
$$

但是，值得注意的是，这样的投影变换是非线性的。3D高斯进行这样的变化后，投影得到的形状将不再满足高斯分布，而这点是我们不希望看到的。

注意到非线性变换是局部线性的，而这种局部的线性变换可以使用其雅可比矩阵表示：

$$
J=\left[\begin{array}{ccc}
f_{x} / t_{z} & 0 & -f_{x} t_{x} / t_{z}^{2} \\
0 & f_{y} / t_{z} & -f_{y} t_{y} / t_{z}^{2} \\
0 & 0 & 0
\end{array}\right]
$$

因此，我们粗略地认为某一3D高斯的投影变换可以使用其均值点的雅可比矩阵等效，故投影后仍能保持2D高斯的性质，协方差$\Sigma'$为：

$$
{\Sigma}^{\prime}=JW\Sigma W^T J^T
$$

终于，我们得到了投影后的2D高斯，其均值为$\boldsymbol{t}$，协方差为$\Sigma^{\prime}$，其第三维代表其深度，在排序时会用到，其他时候可以不用考虑。

最后把数据代入渲染方程(9)就可以顺利算出每个像素的颜色了！

## 6 损失函数

$$
\mathcal{L}=(1-\lambda) \mathcal{L}_{1}+\lambda \mathcal{L}_{\text {D-SSIM }}
$$

方法的损失函数比较简单，为L1_loss和D-SSIM loss的加权和，权重为超参数。
