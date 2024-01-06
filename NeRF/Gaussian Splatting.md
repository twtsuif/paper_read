# Gaussian Splatting

一种**3D表示**，NeRF的替代品。



## 总结

**主要特点：渲染速度快。**

原因：这种表示本身，以及自定义CUDA核函数的定制渲染算法。



**不涉及任何神经网络**，甚至连一个小的MLP都没有。

想法起源于Surface Splatting，一种传统方法。



## 3D表示

millions级别的点，每个点3D高斯分布。每个场景有自己的参数。

> 高斯的均值，很像一张图片。



每个3D高斯，由以下内容参数化

- 均值，可以解释为3D坐标xyz
- 协方差矩阵
- 不透明度，由sigmoid映射成0-1
- 颜色，RGB或SH系数



协方差，被设计为各向异性，意味着3D点可以是沿空间任何方向旋转和拉伸的椭球体。

可能需要9个参数，但并不能被直接优化，因为只有当它是半正定矩阵时，才有意义。

所以执行协方差矩阵的特征分解 $RSS^TR^T$。

可以理解为椭球的配置：

- S是对角缩放矩阵，有3个参数用于缩放。
- R是用四元数表示的3x3旋转矩阵。



高斯函数的美妙之处

根据协方差，每个点有效代表空间中接近均值的**有限区域**。

又具有**理论的无限范围**，每个高斯都是在整个3D空间上定义，可以对任意点评估。所以，优化过程允许梯度从长距离流动。

任意一个3D高斯i，对空间中任意一点p的影响公式：

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-06/00dcc5c697bf82198fd319922a44b0e1--dd55--image-20240106093408983.png" alt="image-20240106093408983" style="zoom:67%;" />

看起来很像多元正态分布的概率密度函数，忽略了归一化项，使用不透明度加权。



## 成像与渲染

事实上，NeRF与Gaussian Splatting具有相同的渲染模型。

NeRF：<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-06/e3fa32bd363368431537b80300d600e0--b036--e3fa32bd363368431537b80300d600e0--d2e9--image-20240106095316697.png" alt="image-20240106095316697" style="zoom: 50%;" />Gaussian：<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-06/553eb1d14755efb8effebdb85600ca69--9b61--image-20240106095322891.png" alt="image-20240106095322891" style="zoom:50%;" />

二者只有α的计算方式不同，这是GaussianSplatting高效的基础。

$f^{2D}$是上方公式的投影，3D点及其投影都是多元高斯分布，所以可以投射到2D计算对其他像素的影响。

协方差矩阵和均值需要投影到二维，这里使用EWA splatting推导。

投影的整个过程仍然是可微的。



渲染过程更加轻量级

- 对于给定的相机，每个3D点的f(p)可以提前投影到2D中。混合周围的像素时，无需一遍又一遍地重投影。
- 2D高斯直接混合到图像上。
- 无需采样。
- 在GPU上排序阶段每帧完成一次。

> 这个混合是什么意思？

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-06/ad7678431077defa952ad2584687fae0--b55d--image-20240106100844733.png" alt="image-20240106100844733" style="zoom:67%;" />

NeRF需要采样，而GaussianSplatting只需要混合一系列的高斯。



**排序算法**

按深度对3D点进行排序，计算透射率。

按照图块对他们进行分组，将每个像素的加权和限制为相关的点。

分组是使用简单的 16x16 像素图块来实现的，如果高斯函数重叠多个单一视锥体，则它可以落在几个图块中。

**通过排序，每个像素的渲染可以简化为来自像素所属图块的预排序点的 α 混合。**

> alpha混合是水平的混合吗？



## 优化

不可能从一堆乱点中获得看起来像样的图片，会得到各种尖状伪影。

良好的渲染：**好的初始化**、**可微分优化**和**自适应致密化**。



**初始化**

使用SfM生成的初始点云，随机初始化也可以，但有潜在的损失风险。

协方差矩阵被初始化为各向同性，以球体开始，实现对场景的覆盖。



**拟合**

SGD。



**自适应致密化**

每隔一定步数启动一次，解决重建不足和过度重建的问题。

分割具有很大梯度的点，删除α值收敛到非常低的点。



## view-dependant的颜色

学习view-dependant的颜色，SH最初在Plenoxels中离散的3D体素中被提出。

视图依赖是一个很好的属性，提高渲染质量，允许非朗伯效应。

当然这不是必须的，可以使用简单的RGB。

球体表面定义的特殊函数，可以对球体上的任何点计算这样的函数并获得一个值。

对于每个 3D 高斯，我们希望学习正确的系数，以便当我们从某个方向查看这个 3D 点时，它会传达一种最接近地面真实颜色的颜色。

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-06/be2d9f12511fa7c3f22c781c1e7199b9--68c3--image-20240106103246084.png" alt="image-20240106103246084" style="zoom:67%;" />



## 缺点

Despite the overall great results and the impressive rendering speed, the simplicity of the representation comes with a price. The most significant consideration is various **regularization heuristics** that are introduced during optimization to guard the model **against “broken” Gaussians**: points that are too big, too long, redundant, etc. This part is crucial and the mentioned issues can be further amplified in tasks beyond novel view rendering.

The choice to step aside from a continuous representation in favor of a discrete one means that **the inductive bias of MLPs is lost**. In NeRFs, an MLP performs an implicit interpolation and smoothes out possible inconsistencies between given views, while 3D Gaussians are more sensitive, leading back to the problem described above.

Furthermore, Gaussian splatting is not free from some well-known **artifacts present in NeRFs** which they both inherit from the shared image formation model: lower quality in less seen or unseen regions, floaters close to an image plane, etc.

The file size of a checkpoint is another property to take into account, even though novel view rendering is far from being deployed to edge devices. Considering the ballpark number of 3D points and the MLP architectures of popular NeRFs, both take **the same order of magnitude of disk space**, with GS being just a few times heavier on average.