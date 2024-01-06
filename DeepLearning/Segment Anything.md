# Segment Anything



## 架构图

分割任务的**基础模型**。

三个组件：**任务**、**模型**和**数据**。

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-05/4911be0bbf5cd71c29d58423790f4d22--5ce0--image-20240105140217529.png" alt="image-20240105140217529" style="zoom: 80%;" />

> Q：数据引擎是什么意思？模型为什么给数据注释，这是循环的吗？
>
> 好像是数据集的意思。



## 关键信息

迄今为止最大的分割数据集：1000万张图片。

因为模型被设计为promptable的，所以有zero-shot能力。

> 为什么呢？



## 读读Introduction

参数量大的基础模型可以**泛化**出训练数据之外的分布，这种能力通常通过prompt engineer实现。

CV中也对基础模型进行了探索，比如CLIP，使两种模态对齐。

SegmentAnything的目的：**使用prompt engineer，在新的数据分布上，解决一系列下游分割问题**。



**三个思考**

Q：什么样的任务能够支持zero-shot？

这个任务要足够通用，可以提供强大的预训练目标，支持广泛的下游任务。

Q：模型要求？

支持灵活的prompt，实时输出mask。

Q：我们是如何收集数据的？

使用高效的模型来帮助我们收集数据，使用新收集的数据来改进模型，反复迭代。



> 训练和预测的区别？
>
> 我们使用可提示的分割任务作为**预训练目标**，并通过**提示工程解决下游任务**。



**模型**

不同的prompt可以重复利用image embedding。

为了能够感知歧义，为单个提示生成多个mask。

架构如上图所示。



**数据引擎**

三个阶段：人工协助、半自动和全自动。

- 传统打码。
- 自动为标注对象生成mask，人工打码其余部分。
- a regular grid of foreground points自动生成。



**实验结果**

通常仅低于真值。

下游任务：边缘检测、对象建议生成、实例分割、文本到掩码。



## Task

从大语言模型获得灵感，下一个词的预测任务用于预训练，prompt用来解决下游任务。

> LLM是这样的吗？不是把prompt一起当作整体，预测下一个词吗？



### 预训练

对于每个训练样本，模拟一系列的prompt，将mask与真值比较。

专门的模型和训练损失，将在模型中讨论。



### 相关任务

分割是一个广阔的领域：交互式分割、边缘检测、超像素化、目标proposal生成、前景分割、语义分割、实例分割、全景分割。

我们的工作与多任务分割不同。可以作为更大系统中的**某个组件**，来在推理时执行新的、不同的任务。

比如为了执行实例分割任务，我们的模型可以结合现有的目标检测器。

类似于其他基础模型的使用方式。



## Model

### 架构

![image-20240105215127545](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2024-01-05/cc871a778c1c7acee6c69c8663e24c1f--bb0f--image-20240105215127545.png)

重量级图像编码器，可以被prompt高效查询，接近实时速度生成mask。

灵活的prompt编码器。

快速的mask解码器。



建立在**Transformer视觉模型**之上。

模型的整体设计很大程度上出于**效率的考虑**。



**图像编码器**

MAE预训练的ViT。



**Prompt编码器**

两种prompt：像点、框、文本这样**稀疏**的和像masks一样**密集**的。

使用**可学习embedding的位置编码**来表示点和框，CLIP中现成的**文本encoder**来表示自由格式文本。

密集prompt，使用卷积embedding，并逐元素对图像embedding求和。



**Mask解码器**

图像embedding、prompt embedding和一个输出token，映射到一张mask。

> 输出token是啥？	嗷，下面有。

受一些工作的启发，对Transformer Decoder进行修改，然后接一个动态mask预测头。

Decoder block在两个方向上(prompt2image和image2prompt)使用prompt自注意力和交叉注意力，来更新所有embedding。

两个block后，对图像embedding上采样，使用MLP将输出token映射到动态线性分类器，计算每个图像位置的mask概率。



### 歧义

我们发现3个掩码输出足够解决最常见的情况：整体、部分、子部分。

训练期间，仅反向传播masks上的最小损失。



### 损失和训练

使用focal loss和dice loss的线性组合。

混合的prompts来训练。



### 具体细节

博客：https://blog.csdn.net/qq_43406895/article/details/129385513