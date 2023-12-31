# Denoising Diffusion Probabilistic Models



![image-20230702134812342](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-02/ae00db3f6769e4b8b8c98e7228885392--f318--image-20230702134812342.png)

这篇论文中的有向图模型



## Abstract

非平衡热力学 启发的 潜变量模型

扩散概率模型 和 Langevin动力学的去噪分数匹配 创新结合设计 权重变分边界

自然承认的渐进式有损压缩方案 可解释为自回归编码器的推广

无条件CIFAR10数据集 9.46的Inception分数 3.17的FID分数

256*256 LSUN上 与ProgressiveGAN类似的样本质量



## Introduction

各种深度生成模型在各种数据形式上展示了高质量样本

GAN、自回归模型、flows、VAE

有一篇文章[53]介绍了扩散概率模型的进展

扩散概率模型 简称扩散模型 使用变分推理来训练的 参数化的马尔科夫链 在有限的时间产生与数据匹配的样本

马尔科夫链的转换被学习 来反向扩散过程 马尔科夫链在相反方向上逐渐向数据添加噪声 直到信号被破坏

扩散包含少量噪声时，将采样链转变为条件高斯，允许特别简单的神经网络参数化



扩散模型定义简单 训练高效 但没有证据表明可以生成高质量样本 我们证明了

扩散模型的某些参数化 揭示了 训练期间多个噪声水平上的去噪分数匹配 和 采样期间退火的Langevin动力学 的等价性



尽管质量很高，但不具有竞争性的对数似然

然而我们的对数似然确实比 基于能量模型和分数匹配 产生的 退火重要性采样的大型估计 要好

模型的大部分无损码长都被用来描述难以察觉的图像细节

我们使用有损压缩语言进行了更精细的分析，并表明扩散模型的采样是一种渐进解码，类似沿位顺序的自回归解码



## Diffusion models and denoising autoencoders

扩散模型看起来可能是一种受限制的潜在变量模型，但在实现中允许着很大的自由度

必须选择 正向过程的方差beta t 和 逆向过程的模型架构和高斯分布参数化

为了指导我们的选择，在扩散模型和去噪分数匹配之间建立了显式连接 为扩散模型提供了简化的权重变分边界目标



### 前向过程和L t

我们忽略了beta t可以通过参数重整化来学习，而是将它们固定为常数

所以在我们的实现中 近似后验q没有可学习的参数 Lt在训练期间是一个常数 可以被忽略



### 后向过程和L 1:t-1



### 数据缩放、逆向过程和L0

假设图像由0-255的整数组成 线性缩放至-1到1，确保逆向过程 操作在 从标准正态先验p(xT)开始的 一致地缩放输入 

为了获取离散对数似然 我们将最后一项设置为从高斯![image-20230630161855261](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/792d6f73795a4f5a312dedd80e872877--f7f1--image-20230630161855261.png)导出的独立离散解码器

![image-20230630161952325](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/c05f07a494939ce86482762459074ad5--cf88--image-20230630161952325.png)

合并为一个更强大的解码器 例如条件自回归模型 将是很简单的

与VAE解码器和自回归模型中使用的离散连续分布类似，我们在这里的选择是，确保变分界限是离散数据的无损码长，无需向数据添加噪声或合并缩放的雅可比行列式对数似然运算

采样结束时 我们无噪声地显示![image-20230630162452660](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/5d9c266f9050fb23e0a13506e897b153--1c31--image-20230630162452660.png)



### 简化训练目标

根据上述逆过程和解码器，从公式推导出来的项组成的变分界限，对于theta明显可微，并且可用于训练

然而我们发现对变分界限的以下变体进行训练有利于样本质量![image-20230630172303049](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/d26b15938e368496cdb8779cb15b4aa0--d45b--image-20230630172303049.png)

t=1的情况对应于L0，离散解码器中的积分近似为高斯概率密度 乘以 bin宽度，忽略方差和边缘效应

t>1的情况对应于方程的无权重版本，类似于NCSN去噪得分匹配模型使用的损失权重





## Experiment

我们为所有实验设置T=1000，以便采样期间所需的神经网络评估量与之前的工作相匹配

将前向过程的方差从beta1=0.0001线性增加到betaT=0.02的常数

这些常数设置较小，确保反向和正向过程具有大致相同的函数形式，保证xt处的信噪比尽可能小   在我们的实验中每维度L t≈0.00001位

为了表示逆向过程，我们使用类似于无掩蔽PixelCNN++的U-Net主干网，并组归一化

参数是跨时间共享的，这是使用Transformer正弦位置嵌入，指定给网络的

我们在16*16特征图分辨率下使用自注意力



### 样品质量

![image-20230630174838998](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/27201f79f711a77ebff12a7a5947e8b4--dcb4--image-20230630174838998.png)

Inception分数、FID分数和负对数似然(无损码长)

我们的无条件模型比文献中大多数模型(包括类条件模型)实现了更好的样本质量

我们的FID分数是根据训练集计算的，标准做法



我们发现正如预期的那样，在真实变分界限上训练的模型比在简化目标上训练产生更好的码长，但后者产生最好的样本质量

CIFAR10、CelebA-HQ、LSUN样本



### 逆向过程参数化和训练目标消融

![image-20230630180351333](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/878b88250a50acb2261b80a79d7efc86--95d2--image-20230630180351333.png)

我们发现预测u~的基线选项只有在真正的变分界限上训练才能很好地工作，而不是在无权重均方误差上训练

还发现与固定方差相比，学习逆向方差(通过将参数化对角线合并到变分界限中)会导致训练不稳定且样本质量较差

正如我们提出的，当在固定方差的变分界限上训练时，预测Epsilon与预测u~的表现大致相同，但在使用我们的简化目标进行训练要好很多



### 渐进式编码

表1还显示了我们的CIFAR10模型的码长，训练和测试之间的差距最多为每维度0.03位，这与其他基于可能性的模型报告相当，表明我们没有过拟合

虽然优于能量模型和退火匹配的大估计，但是与其他基于可能性的生成模型不具有竞争力



由于我们样本仍然是高质量的，所以我们得出结论，扩散模型具有归纳偏差，使得它们成为出色的有损压缩器

将变分约束项L1+...+Lt作为速率，而L0视为失真，超过一半的无损码长描述了难以察觉的失真



#### 渐进式有损压缩

我们可以进一步探讨模型的速率失真行为，通过引入渐进有损编码

![image-20230630184355355](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-30/227756143f8123b4c6c2dd8f7f464394--1aa6--image-20230630184355355.png)

$$

$$

