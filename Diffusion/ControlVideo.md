# ControlVideo





![image-20230619080211098](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-20/203c0e43339a0a6c452d36d4cccbbe10--934c--203c0e43339a0a6c452d36d4cccbbe10--a2b2--image-20230619080211098.png)

沿时间轴膨胀来使ControlNet适应视频的counterpart



## 架构图

![image-20230619085555711](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-20/905c6bb68d3e5f37b8611715c72fca89--ce6f--905c6bb68d3e5f37b8611715c72fca89--e49d--image-20230619085555711.png)

外观一致性：将完全跨帧交互添加到自注意力模块，使ControlNet适应视频counterpart

结构闪烁：交叉帧平滑器 通过交叉的插值来平滑所有帧间转换



## Abstract

无需训练 可控制 文本到视频

时间模型的训练代价太大，视频模型仍然落后文本驱动

外观不一致和结构闪烁，特别是长视频

改编自ControlNet，利用输入运动序列的粗结构一致性，引入三个模块来改进视频生成

- 在自注意力模块加入完全跨帧交互，来确保帧间外观一致性

- 对交替帧进行帧插值的交织帧平滑器，减轻闪烁效应

- 分层采样器分别合成具有整体一致性的每个短片段，有效生成长视频

超越了广泛的motion-prompt pairs

几分钟内在2080Ti生成短视频和长视频



## Introduction

在自然世界建模高维复杂视频分布

大量高质量视频和计算资源



基于T2I模型的可控T2V，生成基于文本描述和运动序列的视频

运动序列的粗时间一致性



最近的研究探索了利用ControlNet或DDIM倒置的结构可控性来生成视频

而不是独立地综合所有帧，通过更稀疏的跨帧注意力代替原始的自注意力来增强外观一致性

问题：1.帧外观不一致，2.大视频的伪影，3.帧过渡的结构性闪烁

对于1和2，他们稀疏的跨帧机制增加了自注意力模块中query和key之间的差异

对于3，输入的运动序列只提供视频的粗略结构，不能在连续帧之间平滑过渡



我们提出：无需训练的可控视频来实现高质量和一致的可控T2V，交叉帧平滑器来提高结构平滑度

直接继承ControlNet的体系结构和权重，通过完全跨帧交互，扩展自注意力，来适应视频的counterpart

全跨帧交互：将所有帧连成串联成更大图像

交叉帧平滑器，通过在选定的有序时间步中，交叉插值，来降低闪烁

每个时间步的运算，通过插值中间帧，来平滑交叉的三帧片段。连续两个时间步的组合，平滑整个视频

平滑操作只在几个时间步执行，所以通过后续的去噪步骤可以保持插值帧的质量和个性



实现长视频合成，引入分层采样器来产生具有长期一致性的分离短片段

长视频分割成多个具有所选关键帧的短视频片段，预生成关键帧，带有完全跨帧注意力，实现长期一致性

以关键帧对为条件，根据全局一致性依次合成相应的中间短视频片段



我们在广泛收集的motion-prompt pairs上实验，由于高效设计(xFormers实现和分层采样)，生成速度快



贡献：

- ControlVideo，包括完全跨帧交互、交错帧平滑器、分层采样器
- 完全交叉注意力更高视频质量，交叉帧平滑器减少闪烁，分层采样器高效生成



## Background

### LDM

潜在扩散模型是扩散模型的高效变体，将扩散过程应用在潜在空间而不是图像空间。

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/2b16428253d9139a30a7a0f7bd98621f--a860--image-20230619091721824.png" alt="image-20230619091721824" style="zoom:80%;" />

编码器E来压缩图像到浅码，解码器D来重建图像

在DDPM公式中学习图像的浅码分布 z~p，包括前向和后向过程

前向扩散过程在每个时间步逐渐添加高斯噪声，获取z

反向去噪过程将上述扩散过程反向预测更少的噪声

u和Σ被去噪模型带有可学习参数θ实现

生成新样本，我们开始z~N，使用DDIM采样来预测z(t-1)



### ControlNet

使SD在T2I期间支持更可控的输入条件，如深度图、姿态、边缘等

与SD相同的U-Net架构，微调其权重以支持具体任务的条件



![image-20230619091259684](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-20/e7a61f564bfe6611875632c9187ec4ea--b2fe--e7a61f564bfe6611875632c9187ec4ea--5f95--image-20230619091259684.png)转化为![image-20230619091317699](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-20/2567b5ca3ba9f838806ac34b7ee5ab50--194c--2567b5ca3ba9f838806ac34b7ee5ab50--0b14--image-20230619091317699.png)

c表示附加条件

分别SD和ControlNet的U-Net架构，前者称为主U-Net，后者称为辅U-Net



## ControlVideo

给定运动序列c和文本提示t生成长度为n的视频



### 完全跨帧交互

T2I扩展到T2V最大的挑战：时间一致性

利用ControlNet的可控性，运动序列可以在结构上提供粗层次的一致性

即使使用相同的初始噪声，单独使用ControlNet生成所有帧也会导致外观上的严重不一致

我们连成一个大图像，这样内容可以通过帧间交互来共享

考虑SD中的自注意力是由外观相似性驱动的，我们提出，通过添加基于注意力完全跨帧交互，来增强整体一致性。



从沿时间轴的稳定扩散中膨胀主U-Net，同时保证辅U-Net不受ControlNet影响

使用1x3x3的核来代替3x3，将2D卷积层直接转化为3D卷积层

![image-20230619095108481](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/33f796bea1921e356141f5f2f47dfa06--f7e5--image-20230619095108481.png)

添加跨所有帧的交互来扩展自注意力

![image-20230619095214178](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/8a8e624d1ee39598fbfe1b0d1230ed1d--d2f7--image-20230619095214178.png)



之前工作通常使用更稀疏的跨帧机制来代替自注意力，所有帧只关注第一帧

短视频生成时(<16帧)，完全跨帧注意力带来更少的内存和可接受的计算负担



### 交叉帧平滑器

![image-20230619095918721](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/9cea09d8900840c9e9787bc9c44d0396--e6ad--image-20230619095918721.png)

我们的主要想法是通过插值中间帧来平滑每个三帧片段，以交错的方式重复，来平滑整个视频。

以有序的timesteps对预测的RGB帧执行

在每个timestep的操作内插偶数帧或奇数帧，以平滑相应的三帧片段

通过这种方式，来自两个连续timestep的平滑三帧片段，重叠在一起降低整体的闪烁

在timestep=t应用交叉帧平滑器之前，根据zt预测干净的视频浅码

![image-20230619103145719](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/23767f96ee3443a9b063348341ffb3ab--cc9a--image-20230619103145719.png)

投影zt到RGB视频后，交叉帧平滑器转化为更平滑的视频xt。基于平滑后的视频潜码，我们计算了更少的噪声潜码z(t-1)

![image-20230619104411770](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/32cd8e7d3ed86352add1350338650c63--7b12--image-20230619104411770.png)

上述处理仅在选定的中间时间步骤中执行，有两个优点

新的计算负担可以忽略不计，通过以下去噪步骤很好地保留了插值帧的个性和质量



### 分层采样器

视频扩散模型需要保持帧间交互的时间一致性，需要大量的GPU内存和计算资源，尤其长视频。

为了促进更高效和一致的长视频，我们使用分层采样器以clip-by-clip的方法生成长视频。

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/b173ca9b361793edbd61b273a48aa637--c213--image-20230619105259007.png" alt="image-20230619105259007" style="zoom:80%;" />

分割成带有所选关键帧的多个短视频片段

预生成带有完全交错帧注意力的关键帧



## Experiment

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/66b0d00dc92150c85dfe070eaa81e840--dc17--image-20230619103444275.png" alt="image-20230619103444275" style="zoom:67%;" />

基于深度图和精明边缘的定性比较

![image-20230619104753861](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-19/5fa6edd1f28c980bb4ec810deb912eb4--b752--image-20230619104753861.png)





## Question

外观一致性？

粗时间一致性？粗略结构？

DDIM？

