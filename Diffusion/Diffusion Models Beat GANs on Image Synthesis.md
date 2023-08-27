# Diffusion Models Beat GANs on Image Synthesis





## Abstract

我们在一系列的消融实验中找到了更好的架构，实现无条件图像合成

对于条件图像合成，通过分类器指导进一步提高样本质量

一种简单、计算高效的方法，使用分类器的梯度来权衡多样性和保真度

在ImageNet 128×128上实现2.97的FID  256×256 4.59 512×512 7.72 

即使每个样本只有25次前向传播，也可以与BigGAN-deep相比，同时保持更好的概率分布

发现分类器指导与上采样模型结合得很好，将256×256的FID提高到3.94，将512×512的提高到3.85



## Introduction

还有一个评价指标Precision

GAN虽然是最先进  但一些指标并不能完全捕获多样性，事实证明捕获的多样性少于最先进的基于likelihood的模型

GAN很难训练，如果没有仔细选择超参数和正则化器，就会崩溃



虽然GAN拥有最先进的技术，但缺点使得它难以扩展到新领域

所以为了实现基于likelihood的模型实现类似GAN的样本质量，人们已经做了很多工作，但是在视觉样本质量方面仍然存在不足

除了VAE外，从wall-clock time来看，采样速度比GAN慢



扩散模型基于likelihood，生成图片，并且提供理想的属性，例如分布覆盖范围、固定训练目标和易于扩展性

在LSUN和ImageNet等困难的生成数据集中仍然落后GAN

Nichol和Dhariwal发现这些模型随计算量增加而提升可靠性，即使在困难的数据集上，使用上采样也可以生成高质量样本

然而仍然无法与最先进的BigGAN-deep竞争



我们假设扩散模型和GAN之间的差距至少来自两个因素：

GAN已经被大量探索和完善

GAN能够以多样性换保真度，产生高质量样本，但不能覆盖整个分布



我们的目标是为扩散模型带来这些好处，改进模型架构，设计一种多样性换保真度的方案



第二部分，我们简要介绍了基于Ho等人的扩散模型背景，Nichol和Dhariwal 以及Song等人的改进，我们描述了我们的评估

第三部分，我们介绍了简单的架构改进，这些改进极大地提高了FID

第四部分，描述了在采样过程中使用分类器梯度来指导扩散模型，我们发现可以调整单个超参数(分类器梯度的scale)，牺牲多样性来换取保真度

我们可以将这个梯度scale因子增加一个数量级，而无需获得对抗性示例

第五部分，展示了具有改进架构的模型在无条件图像合成、分类器指导下有条件图像合成上实现了最先进的技术

我们还将改进的模型与上采样栈进行比较，发现这两种方法提供了互补的改进，将他们组合起来在256×256和512×512上给出了最佳结果



## Background

扩散模型，每个时间步t对应一定的噪声水平，xt可认为x0与一些噪声的混合，信噪比由时间步长t确定

本文的其余部分中，我们假设噪声是从对角高斯分布中提取的，对于自然图像非常有用，简化了各种推导

将模型参数化为函数$\epsilon_\theta(x_t,t)$，该函数可以预测噪声样本$x_t$的噪声组件

训练目标就是真实噪声与预测噪声之间的简单均方误差损失



如何从噪声预测器中采样并不是显而易见的，逐步预测xt-1

在合理的假设下，我们可以对给定xt的xt−1分布 pθ(xt−1|xt) 进行建模，对角高斯 N (xt−1; μθ(xt, t), Σθ(xt, t))

其中平均值 μθ(xt, t) 可作为 εθ(xt, t) 的函数计算，方差可以固定为已知常数或使用单独的神经网络头学习，扩散步骤T足够大时，两种方法都会产生高质量样本

Ho观察到，实践中，简单均方误差损失Lsimple比变分下界Lvlb效果更好，后者可以通过将去噪扩散模型解释为VAE来推导

还指出，以此为目标训练，并使用相应的采样程序，相当于Song和Ermon的去噪分数匹配模型，使用朗之万动力学，从经过多个噪声级训练的去噪模型中采样

我们经常使用扩散模型作为简写，来指代这两类模型



### Improvements

Nichol和Dhariwal建议将方差sigma参数化为神经网络，输出v并插值进去

![image-20230712133507853](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/4fabfe69ef2bb2386613d59dc85abf00--bce9--image-20230712133507853.png)

βt 和 ̃ βt是 Ho 等人的方差，对应于逆向过程方差的上限和下限

提出了一个混合目标，使用Lsimple+λLvlb加权和来训练均值和方差，可以使用更少的步骤采样

我们采用这个目标和参数化



Song等人提出DDIM，制定了一种替代的非马尔可夫噪声过程，与DDPM具有相同的forward marginals，但允许通过反向噪声的方差来产生不同的反向采样器

将此噪声设置为0，提供了一种将任何模型εθ转换为潜在变量到图像的确定性映射，更少的采样步骤

少于50个采样步骤时，我们采用这种方法



### Sample Quality Metrics

IS，衡量模型捕获完整ImageNet类分布的程度，同时处理单个类，缺点：不会奖励覆盖整个分布，或捕获类别内的多样性

FID提供了Inception-V3潜在空间中两个图像分布之间距离的对称测量

Nash等人提出sFID，作为FID的一个版本，使用空间特征而非标准的池化特征

捕获空间关系，奖励具有连贯的高级结构的图像分布

Kynkäänniemi 等人提出了Improved Precision and Recall指标

分别测量样本保真度作为模型样本落入数据流形的比例（精度），多样性作为数据样本落入样本流形的比例（召回率）



使用FID作为整体样本质量比较的默认指标，捕获了多样性和保真度

Precision或IS衡量保真度，并使用 Recall 来衡量多样性或分布覆盖范围

在与其他方法进行比较时，我们尽可能使用公共样本或模型重新计算这些指标

这有两个原因：首先，一些论文与不容易获得的训练集的任意子集进行比较；

其次，细微的实现差异可能会影响最终的 FID 值。为了确保比较的一致性，我们使用整个训练集作为参考批次，并使用相同的代码库评估所有模型的指标



## Architecture Improvements

Ho等人介绍了扩散模型的U-Net架构，Jolicoeur-Martineau等人发现与之前用于去噪分数匹配的架构相比，可以显着提高样本质量

U-Net使用单头16x16分辨率的全局注意力层，并将timestep embedding的投影，添加到每个残差块



我们探索以下架构变化

- 增加深度与宽度
- 增加注意力头
- 在32x32、16x16、8x8分辨率下使用注意力，而不仅仅是16x16
- 使用BigGAN残差块对activations进行上采样和下采样
- 使用根号2分之一，重新调整剩余连接



对于所有的比较，在ImageNet 128x128上训练模型，批量大小256，250个采样步骤

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/b040b39b6e2dc036532e5fd4946906d7--7c8e--image-20230712140713766.png" alt="image-20230712140713766" style="zoom:80%;" />

各种架构改变的消融实验，在700K和1200K迭代中进行评估



除了重新调整剩余连接之外，所有其他修改都可以提高性能并具有积极的复合效果

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/4c713b142fc6d7eafd3c40ced3240e80--b875--image-20230712153323883.png" alt="image-20230712153323883" style="zoom:80%;" />

消融各种架构变化，将 FID 显示为挂钟时间的函数。为了提高效率，FID 评估了超过10000个样品，而不是 50000个样品。

虽然增加深度有助于提高性能，但增加训练时间，并且需要更长的时间才能达到与更广泛的模型相同的性能，因此我们选择不在进一步的实验中使用此更改



还研究了其他更适合Transformer架构的注意力配置。我们尝试将注意力头固定为常数，或固定每个头的通道数。

对于架构的其余部分，我们使用 128 个基本通道、每个分辨率 2 个残差块、多分辨率注意力和 BigGAN 上/下采样，并训练模型进行 700K 次迭代。

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/672021a0f9f27b8969b58577e2862d81--d1fe--image-20230712153813009.png" alt="image-20230712153813009" style="zoom:80%;" />



### Adaptive Group Normalization

尝试了一个被我们称为自适应组标准化（AdaGN）的层，在组标准化操作之后将timestep和class embedding合并到每个残差块中，类似于自适应实例规范和Film

将该层定义为 AdaGN(h, y) = ys GroupNorm(h) + yb，其中h是第一个卷积之后残差块的中间激活，y = [ys, yb] 是从timestep和class embedding线性投影获得

AdaGN改进了我们最早的扩散模型，默认包含在我们所有的运行中

![image-20230712154456747](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/4fc3f97814f945ba0f69baeec44ce438--42a4--image-20230712154456747.png)

消除将timestep和class embedding投影到每个残差块时使用的逐元素操作，使用 Ho 等人的 Addition + GroupNorm 层替换 AdaGN，FID变差

两种模型都使用 128 个基本通道和每个分辨率 2 个残差块、每个头 64 个通道的多分辨率注意力以及 BigGAN 上/下采样，并接受了 700K 次迭代的训练



本文其余部分，使用最终改进的模型架构作为默认值

每个分辨率有 2 个残差块的可变宽度，每个头有 64 个通道的多头，32、16 和 8 分辨率的注意力，BigGAN 残差块向上和下采样和自适应组归一化，用于将时间步长和类嵌入注入到残差块中



## Classifier Guidance

条件图像合成的GAN还大量使用class标签，通常采用类条件归一化统计数据的形式，以及带有明确设计为类似于分类器 p(y|x)的头部的判别器

Lucic等人进一步证明，类别信息对这些模型的至关重要，这对于标签有限的情况下，生成合成标签很有帮助

探索在**类标签上**控制扩散模型，已经将类信息合并到标准化层

探索一种不同方法：利用分类器 p(y|x) 来改进扩散生成器

Sohl-Dickstein等人和Song等人展示了实现这一目标的一种方法，可以使用分类器的梯度来调节预训练的扩散模型

在噪声图像xt上训练分类器pφ(y|xt, t)，使用梯度 ∇xt log pφ(y|xt, t) 引导扩散采样过程朝向任意类标签 y



首先回顾使用分类器导出条件采样过程的两种方法

然后描述如何在实践中使用此分类器

简洁起见，选择符号pφ(y|xt, t) = pφ(y|xt) 和 εθ(xt, t) = εθ(xt)，指的是每个时间步 t 的单独函数，训练时模型必须以输入 t 为条件



### 带条件的反向去噪过程

为了在标签y上进行调节，只需根据以下条件，对每个转换进行采样

![image-20230712161232621](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/fe56a879d877814bf25b08441984848d--0413--image-20230712161232621.png)

Z是归一化常数，从这个分布中精确采样通常很困难，但Sohl-Dickstein等人表明可以近似为**perturbed 高斯分布**

![image-20230712161635687](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/d5b9dd5e818e2c2f856754d63bbdb593--cebd--image-20230712161635687.png)

我们假设logφ p(y|xt) 与 Σ−1相比具有较低的曲率，这个假设在无限扩散步下是合理的，Σ的绝对值为0

这种情况下，我们可以使用xt=u附近的泰勒展开式来近似logφ p(y|xt) 

![image-20230712193734343](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/0f3a56834c29b7fa1d6e2ad99c00a94d--5512--image-20230712193734343.png)

这得到

![image-20230712193823443](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/739ff26c8d88de8adc15dcc1db0833ae--c7ff--image-20230712193823443.png)

我们可以安全地忽略常数项C4，它对应归一化系数Z

我们发现条件转移算子可以用类似于无条件转移算子的高斯近似，但其均值偏移 Σg

![image-20230712194037320](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/61716faace8ac592e93a4c14df51888b--7526--image-20230712194037320.png)

我们为梯度提供了一个可选的比例因子s



### DDIM条件采样

上述条件推导仅适用于随机扩散采样过程，不能应用到DDIM这样的确定性采样方法

改编自Song等人基于分数的调节技巧，利用了扩散模型和分数匹配之间的关系

如果有一个模型εθxt可以预测添加到样本中的噪声，可以使用它来推导分数函数

![image-20230712195048177](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/b672810fb1c50755948a7d7eb66a784a--a4cb--image-20230712195048177.png)

我们现在可以将其带入 p(xt)p(y|xt)分数函数

![image-20230712195302026](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/786f161cc84f3977f077e9fb74366d7b--8b89--image-20230712195302026.png)

我们可以定义新的epsilon预测，对应于联合分布的分数

![image-20230712195341252](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/c8f5c7cf1f7ff93883df69a47e6044e3--df07--image-20230712195341252.png)

可以使用与常规 DDIM 完全相同的采样过程，但使用修改后的噪声预测 ˆ θ(xt)

![image-20230712195429113](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/7496601bfe94167c992bf925b9b7802f--5d22--image-20230712195429113.png)



### 缩放分类器梯度

分类器架构只是UNet模型的下采样主干，在 8x8 层有一个注意力池以产生最终输出

我们在与相应的扩散模型相同的噪声分布上训练这些分类器，并添加随机裁剪以减少过拟合

训练后，我们使用公式将分类器合并到扩散模型的采样过程中，如算法1



在使用无条件ImageNet模型的初始实验中，我们发现有必要按大于 1 的常数因子缩放分类器梯度

当使用1的尺度时，我们观察到分类器为最终样本的所需类别分配了合理的概率（约 50%），但这些样本在目视检查时与预期类别不匹配

扩大分类器梯度解决了这个问题，分类器的类别概率增加到接近100%

![image-20230712200112935](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/539293aa76ae4378f3a99afdd2895ed3--2b90--image-20230712200112935.png)

左侧scale 1.0		右侧scale 10.0



了解效果，![image-20230712200422434](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/741f271f719c89f3a476e20686d290af--bce4--image-20230712200422434.png)，Z是任意常量

调节过程理论上仍然基于，与 p(y|x)s成比例的重新归一化分类器的分布

当 s > 1 时，该分布变得比 p(y|x) 更尖锐，因为较大的值会被指数放大

换句话说，使用更大的梯度尺度更多地关注分类器的模式，这对于生成更高保真度（但多样性较低）的样本可能是理想的



上述推导中，我们假设底层扩散模型是无条件的，对 p(x) 进行建模。还可以训练条件扩散模型 p(x|y)，并以完全相同的方式使用分类器指导。

![image-20230712200939430](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/dc207e644b58aa67e841a818b1f70011--309a--image-20230712200939430.png)

通过分类器引导，无条件模型和条件模型的样本质量都可以得到极大的提高。

在足够高的规模下，引导无条件模型可以非常接近无引导条件模型的 FID，尽管直接使用类标签进行训练仍然有帮助。

指导条件模型进一步改进 FID。

并且分类器指导以召回率为代价提高了精度，从而在样本保真度与多样性之间进行权衡。



<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/80c4ba4c082ff718c3c37c83aa6286e7--aff3--image-20230712201153832.png" alt="image-20230712201153832" style="zoom:80%;" />

我们看到，将梯度扩展到 1.0 以上可以顺利地权衡召回率（多样性的衡量标准）和更高的精度和 IS（保真度的衡量标准）。由于 FID 和 sFID 取决于多样性和保真度，因此它们的最佳值是在中间点获得的。我们还将我们的指导与图 5 中 BigGAN 的截断技巧进行了比较。我们发现，在用 FID 换取 Inception Score 时，分类器指导严格优于 BigGAN-deep。不太明确的是精度/召回率权衡，这表明分类器引导只是在达到某个精度阈值之前是更好的选择，在此之后它无法实现更好的精度。