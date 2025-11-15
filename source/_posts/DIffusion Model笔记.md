---
title: DIffusion Model笔记
date: 2024-05-30 22:21:55
excerpt: 记录一些Diffusion Model相关的知识点
categories: [技术, 计算机视觉]
tags: [Diffusion Model, 生成模型]
---
# DIffusion Model笔记

## 前置知识

### 1. 贝叶斯公式

$$
P(A|B)= \frac{P(B|A)*P(A)}{P(B)}=\frac{P(AB)}{P(B)}
$$

### 2. 马尔可夫过程

马尔可夫过程（Markov process）是一类随机过程。

对于一阶马尔科夫模型，则有第 $i $ 时刻上的取值依赖于且仅依赖于第$ i−1 $时刻的取值，即:

$$
P(x_i|x_{i-1},...,x_1) = P(x_i|x_{i-1})
$$

### 3. 最大似然估计(Maximum Likelihood Estimation)

在统计学中，最大似然估计（maximum likelihood estimation，MLE），也称极大似然估计，是用来估计一个概率模型的参数的一种方法。最大似然估计在统计学和机器学习中具有重要的价值，常用于根据观测数据推断最可能的模型参数值。

简而言之，当我们有一个固定参数集的模型并且我们对可能生成的数据类型感兴趣时，通常会考虑概率。相反，当我们已经观察到数据并且我们想要检查某些模型参数的可能性时，就会使用似然。

- 概率质量函数（PMF）：参数已知，观察数据。
- 似然函数(Likelihood Function)：数据已知，评估参数。

参考资料：[一文了解最大似然估计(Maximum Likelihood Estimation)](https://mp.weixin.qq.com/s?__biz=MzI1MjQ2OTQ3Ng==&mid=2247604343&idx=1&sn=8659045f8c4279710a205da9303af5e8&chksm=e9e051fcde97d8ea1dc8cc716325c4e7c8ce7da52af653e8462039f51caa7ad7703805d18626&scene=27)

## 扩散模型的数学原理
