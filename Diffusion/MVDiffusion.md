# MVDiffusion

Enabling Holistic Multi-view Image Generation with Correspondence-Aware Diffusion

通过Correspondence-Aware Diffusion实现整体性的**多视图图像生成**



github.io展示页面：https://mvdiffusion.github.io/

展示了生成全景图和多视图D2I



## 架构图

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-07/bb17edb3459cf10117d4dc73a229d4c3--8480--image-20230707113653900.png" alt="image-20230707113653900" style="zoom:80%;" />

多视图对应感知注意力块（MV CA ）用于连接不同的视图

对于超分辨率模块，合并了额外的 SR CA 块来聚合来自低分辨率图像的信息



## 简写名词

correspondence-aware attention CA



## 概括

输入：文本提示+输出图像的视图信息



## Abstract

适用于pixel-to-pixel correspondences的场景，例如给定几何形状(深度图和姿势)的全景图或多视图图像的透视裁剪

先前模型：依赖重复的图像warping和inpatienting，存在**累积误差**

MVDiffusion同时生成所有图像，生成图像带有全局意识，高分辨率+丰富

特别结合correspondence-aware注意力机制，实现跨视图交互，支撑着三个重要的模块

- 生成模块：保持全局correspondence，生成低分辨率图像
- 插值模块：增加图像间空间覆盖范围的密度
- 超分模块：生成高分辨率输出

全景图像，可生成1024*1024

几何条件的多视图图像生成，第一种方法，生成场景网格的纹理贴图



## Introduction

diffusion相关工作

多视图T2I仍然困难：计算效率和多视图一致性

一种常见的方法涉及**自回归生成**过程，通过图像warping和inpatienting，以第 (n-1) 个图像为条件生成第n个图像

累积误差，并且不能处理loop closure，对先前图像的依赖可能会给复杂场景或大视点变化带来挑战



在**透视图象**上使用**预训练的潜在Diffusion模型**同时地生成图像

保留了原始的SD模型，在与不同视图相关的UNet块之间，引入了correspondence-aware注意力机制

支撑着三个模块...

**多视图一致性**或**conditioning**有着pixel-to-pixel correspondence，所以correspondence-aware注意力是适用的



提出了一种**量化多视图一致性**的指标，计算生成图像pairs和自然图像pairs中，重叠区域的PSNR，对比整个测试集中的PSNR评估一致性



贡献

- 多视图图像生成架构，基于透视图像预训练的潜在扩散模型，同时生成一致的多个图像
- 在生成基于几何条件的全景图和多视图图像方面具有最先进的性能
- 评估生成图像的多视图一致性的新指标



## Preliminary

LDM是方法的基础，由三个组件组成：带有encoder$\varepsilon$和decoder$D$的VAE、去噪网络$\epsilon_\theta$、条件编码器$\tau_\theta$

高分辨率图像**x**通过$Z=\varepsilon(x)$映射到低维潜在空间

SD中，下采样因子f=H/h=W/w被设置为8

通过$x^~=D(Z)$将潜在变量变回图像空间

训练目标![image-20230707103953911](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-07/6f9cabe6031b68c11726168142d4332a--6fe5--image-20230707103953911.png)

t 从1到 T均匀采样，去噪网络 εθ 是一个时间条件 U-Net，通过交叉注意机制进行增强，以合并可选的条件编码 τθ(y)，y可以是文本提示、图像等

在采样时，去噪过程会在潜在空间中生成样本，并且解码器通过单次前向传递生成高分辨率图像。采用先进的采样器[22、28、42]来加速潜在样本的采样过程



## MVDiffusion

pixel-to-pixel

- 全景图，图像由覆盖全景视野的N个透视图像生成，像素的一致性pixel-to-pixel correspondence来实现，每个像素对应的homography矩阵
- 多视图深度图像，图像集由给定相机位姿生成，适用深度信息的非投影和投影操作来实现pixel-to-pixel correspondence

在分辨率为图像1/8的潜在特征空间中运行，SD的VAE最终将特征图转为输出图像



### Correspondence-aware Attention

![image-20230707112723120](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-07/9596aece78166b8515fc281330a0782b--0449--image-20230707112723120.png)

多个特征映射之间强制执行correspondence约束，该机制通过考虑源特征图F和N目标特征图$F^l$来运行

对于位于源特征图中位置s的标记，我们基于目标特征图Fl中对应的像素tl，与局部的邻域计算信息

具体来说，对于每个目标像素 tl，我们通过将整数位移 (dx/dy) 添加到 (x/y) 坐标来考虑 K × K 邻域 N (tl)，其中 |dx| < K/2 和 |dy| < K/2

实践中提高质量，我们使用 K = 3 和 9 个点的邻域来生成全景图，并使用 K = 1 来生成几何条件多视图图像

消息m计算

![image-20230707112135868](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-07/9d888bc95b5b4403d88b3819d83c87fe--ce7a--image-20230707112135868.png)

消息计算遵循标准注意力机制，该机制将目标特征像素 {tl*} 的信息聚合到源（s）

关键区别是基于源图像中对应位置 sl* 与 s 之间的 2D 位移（全景）或 1D 深度误差（几何），向目标特征 Fl(tl*) 添加位置编码 γ(·)

在全景生成中，位移提供了局部邻域中的相对位置

在深度到图像生成的情况下，位移提供了有关深度不连续性或遮挡的线索，这对于高保真图像生成至关重要

位移是一个 2D 向量，我们对 x 和 y 坐标中的位移应用标准频率编码，然后连接

目标特征 Fl(tl*) 不在整数位置，是通过双线性插值获得的



### Multi-view Latent Diffusion Model

**生成模块**

高斯噪声初始化输出图像的潜在值，在文本条件下通过SD pipeline生成相应图像

确保多视图一致性，每个U-Net块中引入并微调CA层

**插值模块**

灵感来自最近的VideoLDM

以一对生成的图像“关键帧”为条件，并生成中间图像，CA层添加在每对预置图像之间

生成关键帧特征图或中间特征图以强制多视图一致性

**超分模块**

在每对特征图和/或高分辨率特征图之间添加CA层在 U-Net 块内强制执行多视图一致性



在全景生成任务中，全景图由八个透视图形成，其中水平视场为90度，重叠为45度，生成模块生成八个256×256图像，超分辨率模块升级到八个1024×1024图像

对于深度到图像任务，它涉及综合使用生成和插值模块来生成图像

这些任务所需的数据，特别是输入相机姿态和深度细节，是从 ScanNet 场景中提取的

该过程从生成模块创建关键图像开始，然后通过插值模块对这些图像进行致密化以获得更详细的表示



#### 生成模块

多分支 U-Net 架构中以预测噪声

在每个 U-Net 块中，我们集成了一个配备了CA的Transformer块，后面是一个额外的残差块

为了保留SD的固有功能，如ControlNet所建议，我们将Transformer块的最终线性层和残差块的最终卷积层初始化为零

系统允许为不同的透视图像指定不同的文本提示，从而增强对生成内容的控制

#### 插值模块

在一对“关键帧”之间创建 n 个图像，这些图像之前已由生成模块生成

利用与生成模型相同的 unet 结构和CA权重，具有额外的卷积层，并使用高斯噪声重新初始化中间图像和关键图像的潜在特征

显著特征是关键图像的 Unet 分支以已生成的图像为条件，此条件被合并到每个 Unet 块中

在关键图像的 Unet 分支中，生成的图像与 1（4 个通道）的掩码连接，然后使用零卷积运算将图像下采样到相应的特征图大小

这些下采样条件随后被添加到 Unet 模块的输入中

对于中间图像的分支，我们采取不同的方法

我们将像素值为零的黑色图像附加到零掩码，并应用相同的零卷积运算对图像进行下采样以匹配相应的特征图大小。

这些下采样条件也会添加到 Unet 模块的输入中。

此过程本质上是对模块进行训练，以便当掩码为 1 时，分支重新生成条件图像，而当掩码为零时，分支生成中间图像。

#### 超分模块

首先使用 VAE 编码器将低分辨率图像转换为潜在图像，

然后使用两个具有 3×3 内核的卷积层将通道维度从 4 增加到 320，并获得低分辨率图像特征

通过高斯初始化高分辨率图像latents，并采用比生成模块更轻的 Unet 来生成高分辨率图像，其中 CA 层连接低分辨率图像特征和高分辨率特征图

具体来说，在每个 UNet 编码器块中，我们对这些提取的特征应用一个卷积层以匹配 Unet 通道维度，并引入一个从低分辨率到高分辨率图像的单向对应Transformer块，即图 2 中的 SP CA 块这个额外的对应注意层增强了高分辨率输出的细节和保真度，有效地利用了初始阶段获得的粗略全局结构。



在训练过程中，我们仅在每个 1024 × 1024 高分辨率图像的中心使用 512 × 512 图像块。

在推理过程中，我们将训练后的模型应用于图像的整个范围

这种方法使我们能够有效地处理高分辨率图像，而不会影响超分辨率模型的性能和有效性



#### 训练

首先训练生成模块，然后训练插值，同时它们共享相同的 Unet 和对应感知权重。

超分辨率模块是单独训练的。

对于每个模块，训练过程包括两个阶段。

在第一阶段，我们对稳定扩散 U-Net 模型进行微调，同时冻结在单视图透视图像上训练的 VAE 组件。这种方法允许模型从更广泛的内容中获取知识。

在第二阶段，我们将零卷积的 CA Transformer块集成到网络中，以强制多视图一致性或调节

仅训练附加的 CA 变压器块，而预训练的权重是固定的。



在每个训练步骤中，我们对多视图图像的共享噪声级别 t 从 1 到 T 进行统一采样。我们对生成模块和超分辨率模块都采用以下损失函数，其中 εi θ 是估计噪声，Zit 是第 i 个图像的潜在噪声：

![image-20230707161532666](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-07/b37e0b480d8503ef37242ff4442a1a1e--3c50--image-20230707161532666.png)



## Experiments

### 实施细节

使用了https://github.com/huggingface/diffusers

由一个用于在压缩潜在空间内执行去噪过程的U-Net和一个用于连接图像和潜在空间的 VAE 组成

SD的预训练VAE使用官方权重进行维持，用于在训练阶段对图像进行编码，并在推理阶段将潜码解码为图像

 4 个A6000



### 评估指标

#### 生成质量

FID、IS、CS

#### 多视图一致性

一种**基于像素级相似性**的新指标

多视图图像生成领域仍处于早期阶段，并且没有多视图一致性的通用度量

基于PSNR的新指标

给定多视图图像，计算所有重叠区域之间的 PSNR，然后比较真值图像和生成图像的重叠PSNR。

最终得分定义为生成图像的“重叠 PSNR”与真值图像的“重叠 PSNR”之比。值越高表示一致性越好。