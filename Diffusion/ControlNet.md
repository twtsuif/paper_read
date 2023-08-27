# ControlNet

Adding Conditional Control to Text-to-Image Diffusion Models

在T2I扩散模型中加入条件控制



## 架构图

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-11/2f4f103fd723bda16bcd973bf98b8917--6c1a--image-20230711234916203.png" alt="image-20230711234916203" style="zoom:80%;" />

将ControlNet应用到任意神经网络块

xy是神经网络的深层特征，+是指特征加法，c是额外条件，零卷积是1x1卷积层(权重和偏差都初始化为0)



<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/c1df20d89dda5a79f07e65aa4c64d6cc--e883--image-20230712003420704.png" alt="image-20230712003420704" style="zoom:80%;" />









## 图片



<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-20/17ff5e0c98dee608385d1f2034d5d678--f838--image-20230620125211165.png" alt="image-20230620125211165" style="zoom:50%;" />

使用canny边缘控制SD

canny边缘图作为输入，生成右边的图像不需要原图像

输出通过提示词*高质量、细节、专业* ，不涉及任何有关图像内容和对象名称





## Abstract

控制预训练的大扩散模型来支持额外的输入条件

端到端的方式学习特定的任务条件，即使训练数据集很小

训练ControlNet就像微调扩散模型一样快，可在个人设备上训练

我们提出了，像SD这样的大扩散模型，用ControlNet来增强，如边缘图、分割图、关键点等



## Introduction

T2I，基于prompt的控制真的满足我们需求吗？

大模型能否用于特定任务

构建什么样的框架来处理广泛的问题条件和用户控制

特定任务中，大模型能否保留从数十亿图像中获得的优势和能力



特定任务的可用数据规模没有那么大，需要一种鲁棒的神经网络训练方法避免过拟合

当图像处理任务用数据驱动的解决方案时，大型计算集群并不总是可用的，将大模型优化到特定任务非常重要，要求使用预先训练的权重，以及微调和迁移学习

各种问题有不同形式的定义、用户控制或图像注释，虽然扩散模型可以用程序性的方式进行调节(约束去噪过程、编辑多头注意力激活等)，但这些手工规则的行为由人类指令规定，一些特定问题要求将原始输入解释为对象级或场景级的理解，端到端的学习必不可少



ControlNet将大扩散模型的权重克隆为“可训练副本”和“锁定副本”：锁定副本保留了之前大数据集训练的能力，可训练副本在特定任务的训练集上来学习条件控制

这两个网络块与“零卷积”的独特卷积层相连，卷积权值以学习的方式从零开始增长到优化参数

保留了product-ready权值，训练在不同规模的数据集上都是鲁棒的

零卷积不会给深度特征增加新的噪声，训练与微调一样快



我们用多种不同条件的训练集训练了多个ControlNet，比如canny边缘、Hough线、用户涂鸦、人类关键点、分割图、形状法线、深度等

我们还用小数据集和大数据集来实验ControlNet

我们还展示了一些任务比如depth-to-image，在个人计算机3090Ti训练ControlNet与计算集群训练的商业模型竞争结果



## Related Work

### HyperNetwork与神经网络结构

HyperNetwork起源于一种神经语言处理方法，来训练一个小的RNN来影响一个大的神经网络权重

在使用GAN图像生成和其他机器学习任务中，HyperNetwork也有被报道

受这些想法的启发，[15]提供了一种方法，将小的神经网络附加到SD上，改变其输出图像的艺术风格

[28]提供几个HyperNetwork预训练权重后，该方法得到更多欢迎

ControlNet和HyperNetwork在影响神经网络上有相似之处



使用特殊的零卷积层

早期的神经网络研究，广泛讨论了权重的初始化问题，用高斯分布初始化权重的合理性，初始化权重为零可能带来的风险

[37]讨论了在扩散模型中，缩放几个卷积层的初始权重，来改善训练，该方法与零卷积相似

其他方法，GAN、Noise2Noise等，也提到了零权重



### 扩散概率模型

15年首次被提出，后续通过重要的训练和采样方法进行了改进，DDPM、DDIM、基于分数扩散

图像扩散方法，直接使用像素颜色作为训练数据，高分辨率图像需要考虑节省计算的方法，或者直接使用pyramid-based或multiple-stage方法

本质上使用U-Net

为了减少计算，基于潜在图像，提出了LDM，进一步到SD



### T2I扩散

使用CLIP等预训练语言模型将文本编码为潜在向量

GLIDE文本引导的扩散模型，支持图像生成和编辑

Imagen不使用潜在图像，使用金字塔架构直接扩散像素



### 预训练扩散模型的定制与控制

因为主要都是T2I，增强对扩散模型的控制最直接的通常是文本引导，操作CLIP特征

图像扩散过程，本身可以提供一些功能，实现颜色级别细节变化，SD社区称其为img2img，算法自然支持inpainting作为控制的重要方式

Textual Inversion和DreamBooth使用一小组具有相同主题或对象的图像来定制生成内容



### I2I翻译

尽管与I2I翻译可能有重叠的应用，但本质不同

前者：学习不同域中图像之间的映射

Pix2Pix提出I2I翻译的概念，早期方法以条件生成神经网络为主，Transformer流行起来之后，使用自回归方法取得了成功结果

一些研究表明，多模型方法可以从各种翻译任务中学习鲁棒的生成器

Taming Transfomer能够生成图像和执行图像到图像的翻译

Palette是统一的基于扩散的I2I翻译框架

PITI利用大规模预训练来提高生成结果质量

在sketch-guided扩散等特定领域，[58]基于优化的方法来操作扩散过程



## Method

### ControlNet

操作神经网络的**输入条件**，控制网络的整体行为

复制一个网络参数作为副本，固定原先的参数，复制的θc使用外部向量c训练，避免过拟合和利用大模型的能力

神经网络块通过零卷积层连接，如1x1卷积层

零卷积运算表示为 Z(·;·) ，使用参数 {θz1, θz2} 的两个实例来组成 ControlNet结构

![image-20230711234500820](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-11/6446728f3bd910d7a87f286f89fe2918--bad3--image-20230711234500820.png)

因为权重和偏差都初始化为零，训练第一步，与ControlNet不存在时一致

![image-20230711234718190](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-11/f54e35cc2bb6ed8d457e5262637b2d18--760e--image-20230711234718190.png)

任何进一步优化，都像微调一样快

简单推导零卷积的梯度计算，任何空间位置p和通道索引i，给定输入I，前向传播为

![image-20230712001452388](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/46d535264e7a6a86cdacbfaa0bf9d9f3--6b52--image-20230712001452388.png)

对于Ip,i不为0的地方，梯度变为

![image-20230712001615357](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/c68ce84f3ccb686b06205514280020e8--831f--image-20230712001615357.png)

虽然会导致特征项I的梯度变为0，但权重和偏差梯度不受影响

特征项从数据集中采样的输入数据或条件向量，自然确保了非零I

例如，考虑具有总体损失函数L和学习率βlr不等于0，外部梯度不为0，我们将有

![image-20230712002358680](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/5a08419877a5d179b455620ff1fa072e--21ef--image-20230712002358680.png)

W*是梯度下降后的权重，圈乘是Hadamard product

我们得到![image-20230712002604558](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/44490a76be643f1ad079f440878659aa--0308--image-20230712002604558.png)

获得非零梯度并且神经网络开始学习

通过这种方式，零卷积成为一种独特类型的连接层，以学习的方式逐渐从零增长到优化参数



### ControlNet in Image Diffusion Model

SD 12个encoder 12个decoder 1个middle block

8个是上采样或下采样的卷积层，17个主块，每个块包含4个ResNet和2个ViT，ViT包含多个交叉注意力或自注意力

文本由CLIP编码，时间步由PE编码



与VQGAN类似预处理，转化为更小的潜在图像，这需要ControlNet将**基于图像的条件**，转化为**特征空间**来**匹配卷积大小**。

使用4个卷积层组成的微网络，将图像空间条件ci编码为特征映射



使用ControlNet创建了12个编码块和1个SD中间块的可训练副本，12个块中有4种分辨率，每种有3个块



### Training

去噪可发生在像素空间或从训练数据编码的潜在空间，SD使用潜在图像作为训练域

学习网络εθ来预测添加的噪声

![image-20230712102737415](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/76d1b183f6c2947ffd06d8404e61cabc--1ae5--image-20230712102737415.png)

该学习目标可以直接用于微调扩散模型

训练过程中，随机将50%的文本替换为空字符串，有助于从**输入条件映射**识别语义内容，如边缘图和位姿图

主要是因为：提示对于SD不可见时，倾向于从输入条件映射学习语义信息



### Improved Training

#### 小规模训练

计算能力受限，部分断开ControlNet和SD之间的连接可以加速收敛

当模型结果和条件之间存在合理关联时，断开的链路可以在接下来训练种再次连接，利于精确控制



#### 大规模训练

这种情况下过拟合风险相对较低，先训练足够多迭代次数，解锁所有权重并联合训练



### Implementation

#### Canny Edge

使用Canny边缘检测器，获取3M个edge-image-caption对



## Summary

对扩散模型，加一些条件控制，实现特定的任务，比如深度图到图像、边缘图到图像、位姿关键点图到图像
