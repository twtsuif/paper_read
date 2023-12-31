# IntrinsicNeRF

Learning Intrinsic Neural Radiance Fields for Editable Novel View Synthesis

学习用于可编辑新视图合成的内在神经辐射



## 展示

![image-20230910203122686](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-10/df79dc4e9a198d889d021b998baaec80--6b35--image-20230910203122686.png)

将带有位姿的多视角图片，分解成多视图一致的组件：反射、着色、残差

这种分解支持在线应用：场景recolor、光照变化、可编辑新视图合成



## 架构图

![image-20230910222216342](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-10/f97360a2e8fd8411d8b47d6178ebfcdc--9882--image-20230910222216342.png)

利用无监督先验和反射聚类作为损失函数约束



![image-20230911110651353](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-11/f470584105e0928b70514c3b4cf011cc--57a4--image-20230911110651353.png)

距离感知点采样

NeRF在每次优化中，从大约1024像素的图像随机采一批光线，之间没有建立任何关系

首先随机采样512个点，每个采样点的8个邻居中随机采样剩下的512个点，构造内在分解的无监督约束



## 摘要

现在的逆渲染只能编辑特定的物体，我们引入内在分解，扩展到**房间级场景**

内在分解的本质上是一个**欠约束问题**，提出新的**距离感知点采样**和**自适应反射迭代聚类**的优化方法，无监督的方式

为了更好的聚类，提出了**从粗到细优化的分层**聚类方法，获得更快的**分层索引表示**



## 简介

隐式的神经场需要显式地分解为可编辑属性

有工作提出将逆渲染引入神经渲染，场景被分解为几何、反射、照明

逆渲染从本质上来说有争议并且ill-posed，引入了很多先验，避免物体间相互遮挡、反射和间接光传播，还需要精确的3D表面，所以限制在对象级



内在分解可视为逆渲染的简化，旨在提供可解释的中间表示

简单的解决方案，先用NeRF生成多视图图像，然后执行内在分解，这两步是分开的

我们直接将5D输入，**生成体积密度、与视图无关的反射率和着色、与视图相关的残差**



传统的内在分解和基于NeRF的方法在优化方面存在巨大差距

传统方法：建立与像素相关的约束来优化能量方程

提出距离感知采样法，允许采样点随机，可以在点之间建立局部和全局关系

这样既满足NVS，又能恢复场景固有属性



处理相似反射区域的不一致问题，提出一种带有均值平移的自适应迭代聚类

根据场景本身自适应聚类相似反射的颜色点，比限制了类别数量的KMeans更好

带有体素网格filter的连续更新操作，将相似的反射颜色映射到相同的目标反射颜色，获得每个颜色点的聚类类别

训练期间提出了语义感知的反射率稀疏约束

受SemanticNeRF启发，添加了额外的语义分支，与反射聚类一起产生分层反射迭代聚类和索引方法，从粗到细优化网络



## 相关工作

### 内在图像分解

内在分解是典型的图像层分离问题，将图像分解为反射率、阴影，已经研究了几十年

为了解决这种ill-posed问题，使用了带有优化框架的先验

最近出现了深度学习的方法 无监督



### 内在视频分解

从图像扩展到视频

第一种方法使用运动信息，建立帧之间的相关性来进行后处理

第二种方法优化能量方程，使用先验直接统一图像的局部和全局关系



### 逆渲染

另一种恢复场景基本属性的方法，大致分为经典方法和可微渲染

很多将神经渲染和逆渲染相结合的工作，展示了真实视图合成和对场景底层的一致估计

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-11/4019e99d080ab5579ee086d0497622e8--3c6b--image-20230911105431529.png" alt="image-20230911105431529" style="zoom:80%;" />



## 方法

### 内在神经辐射场

#### 前置知识：内在分解

朗伯假设和灰度着色假设通常用来简化这个逆问题

将图像I表示为，光照不变反射R(I)和光照变化着色S(I)的逐像素乘积 ![image-20230911105907032](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-11/5bede20434b0bf18e895bdb5ff7d675d--059e--image-20230911105907032.png)

现实场景很难满足，内在残差模型引入了视图无关的反射和着色，附加了视图相关的残差项Re(I)，对不满足朗伯假设进行建模，如光泽和金属

![image-20230911110112520](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-11/ca1a9f54a0f29b6843b1d104c0c8c13a--144d--image-20230911110112520.png)

得到空间中每个点的颜色



目标颜色

![image-20230911110522504](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-09-11/3e9ca58154a8698b0bb62b1e1d4bf8ab--56e6--image-20230911110522504.png)





