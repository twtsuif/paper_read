# NeuS

Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction

体渲染学习神经隐式表面进行多视图重建



Github：https://github.com/Totoro97/NeuS



## Abstract

输入：2D图像

现有的表面重建：DVR和IDR，需要前景掩码监督，容易陷入局部最小值，很难用于严重的自闭合或薄表面

NeRF提取高质量表面困难，没有足够的表面约束

SDF的zero-level set表示表面，开发新的体渲染方法，传统体渲染在表面重建固有的几何误差，提出在一阶近似中没有偏差的新公式

数据集：DTU、BlendedMVS



## Introduction

最近的工作将表面表示为SDF或occupancy

使用可微表面渲染将3D对象渲染为图像

IDR无法重建复杂结构，导致深度突然变化，陷入局部最优解，渲染方法仅考虑每条射线的单个表面交点，梯度仅存在于单点，对有效的反向传播来说太局限

因此需要对象掩码作为监督，收敛到有效表面

![image-20230715000946878](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-15/ab04cc4ff430a7cdf1896f1b3cb9b360--16fb--image-20230715000946878.png)

孔洞导致深度剧烈变化，错误预测为蓝色

IDR无法正确重建深度突变的边缘附近的表面



体渲染的优点处理突然的深度变化，考虑了沿着射线的多个点，所有采样点都会产生用于反向传播的梯度信号

NeRF的目的是新视图合成而非表面重建，只学习体积密度场



通过引入由SDF诱导的密度分布，体渲染方式学习隐式SDF



## Question

非朗伯曲面