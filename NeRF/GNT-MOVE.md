# GNT-MOVE

Enhancing NeRF akin to Enhancing LLMs: Generalizable NeRF Transformer with Mixture-of-View-Experts

像增强LLM那样增强NeRF：具有MoVE的可泛化NeRF Transfromer

>Mixture of View Experts



GitHub：https://github.com/VITA-Group/GNT-MOVE



## 架构图

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/2e23c07d2d89f8a03d563146cdbafe2b--0d28--image-20230825142130175.png" alt="image-20230825142130175" style="zoom:80%;" />

左图：目标视图的每条射线，采样的点通过view transformer聚合来自源视图的多视图特征

右图：view transformer中，MoE层嵌入到Transformer块中，Point Token将路由器选择的专家和永久专家处理，强制跨场景的一致性

使用4个MoE嵌入的transformer块，每个MoE有4个专家，共1296，提供足够大和多样化的覆盖范围



## 摘要

泛化到未见过的场景

现有的尝试：端到端的神经化架构。	即使用Transformer那样的高性能神经网络，替换场景表示或渲染模块，将nvs转变为**前馈推理管线**

这些前馈神经化架构仍然不能很好适应不同场景，我们将其与**来自LLM的强大MoE思想**结合

MoE思想通过平衡**更大的模型容量**和**每个实例专业化**，表现出了很好的泛化能力

从最近的泛化NeRF的架构**GNT**开始，首先证明了MoE确实可以插入进去

进一步定制了一个**共享的永久专家**和一个**几何感知的一致性损失**，分别增强**跨场景一致性**和**空间平滑性**

SOTA，表明在zero-shot和few-shot设置中有更好的泛化能力



## 简介

NeRF大多数方法以**后向**方式拟合场景，最近可泛化NeRF形成新趋势：以**前馈**方式			// 前馈和后向是什么意思？

首先通过**学习如何表示场景**和**从不同场景捕获的图像渲染新视图**进行预训练，实现zero-shot

GNT通过统一的、数据驱动的、可扩展的transformer取代了显式的场景建模和渲染功能，通过大规模nvs预训练引入了多视图一致的几何和渲染



泛化性和专业化的两难

- 不同场景的属性(颜色、材质)，需要广泛覆盖不同的场景表示或渲染机制，需要更大的整体模型
- 单个场景通常由独特的自相似外观模式组成，每个场景进行专门化，建模



GNT由**聚合多视图图像特征**的view transformer和**解码点特征来合成新视图**的ray transformer

MoE鼓励**不同的子模型**(即激活专家的组合)针对**不同的输入**，**稀疏地被激活**，因此变得专业化，从而不增加推理成本来扩大模型规模

将MoE引入到view transformer，实验观察到简单地插入不能很好的平衡泛化性和专业化，由于以下两个先验：						// 所以是怎么插入的？

- 跨场景一致性：不同场景相似的属性，应该由相似的专家处理
- 空间平滑性：同一场景的邻近视图应该连续且平滑变化，从而making similar or smoothly transiting expert selection		  // 这是什么句式？

这两个先验由于**自然图像渲染**和**多视图几何约束**

强制执行会导致臭名昭著的MoE崩溃，即不同的子模型学习学习相似的功能，无法捕获不同的专业特征

MoE崩溃已经在很多文献得到解决，但我们需要关注这些解决方案是否会与这两个先验相矛盾



我们调查了MoE两种改进

首先，使用**共享的永久专家**来增强MoE层，在所有情况下都被选择

其次，引入**空间平滑损失**来实现几何感知连续性，鼓励两个空间接近的点选择相似专家，使用采样点之间的几何距离，重新加权专家选择

经验发现，这两种一致性的正则化，与MoE中的专家多样性正则化器配合得很好



## 相关工作

### NeRF及其泛化

NeRF后续工作：新的光线参数化提高渲染质量，显式数据结构或蒸馏提高效率，采用时空方法建模到动态场景

泛化：

一些工作结合卷积编码器，使用基于不同图像特征的相同MLP，来建模不同对象

一些工作采用带有epipolar约束的基于transformer的网络，以前馈方式即时NVS



### Mixture-of-Experts(MoE)

MoE根据某些**学习的或临时的路由策略**，使用子模型的组合，执行依赖于输入的计算

NLP的进展提出了稀疏门控MoE来扩大LLM容量，从而不牺牲每次推理成本，鼓励具有不同功能的模块

在cv领域也很受欢迎，虽然大部分仅关注分类任务



一些工作已经探索了在NeRF中稀疏激活子模块的想法

...



## 前置知识

### GNT

阶段1：view transformer预测每个点的坐标对齐特征，通过聚合来自其相邻视图的极线的信息

阶段2：ray transformer沿着光线，组合逐点的特征来计算光线颜色



给定N个源图像{I}，从目标视图发出的射线上的每个采样点x，公式化为 ![image-20230825141754577](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/5d14ff04d96dcd3f84685fc46234fc5a--17e6--image-20230825141754577.png)

Ⅱ是将x投影到第i个图像平面上，F是基于UNet的小型CNN，在投影图像点处插值特征

V-Trans将所有提取的特征组合为坐标对齐的特征体



ray transformer对预测的token执行均值池化，通过MLP映射到RGB，公式化为 ![image-20230825143104329](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/916c7ec71062fa22d33d1f5f994d12b2--82cb--image-20230825143104329.png)





### MoE

MoE层通常包含一组E专家{f}和一个输出E维向量的路由器R

专家网络是ViT中的多层感知的形式

路由器R专家选择，称为top-K门控

输入token x，输出y可表示为使用路由器选择的前K个专家的总和

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/8712404241b1259f9eeb84d42bd55770--afa5--image-20230825143657755.png" alt="image-20230825143657755" style="zoom:80%;" />



## 方法

设计原则：对原始的GNT进行必要的最小修改



### MoVE：基础管线

GNT利用了unet提取几何、外观和局部光传输信息，V-Trans在R-Trans的潜在空间上，集成特征来评估逐点渲染的参数(占用率、不透明度、反射率)

我们注意到，自然着色属性通常互斥(漫反射和镜面反射)，因此稀疏激活

并且，在渲染引擎中，场景的显示涉及到不同的图形着色器，来处理时空变化的材质

这些观察促使我们将MoE模块插入V-Trans，针对特定的渲染属性专门化不同的组件

我们用稀疏激活的MoE层替换密集的MLP层



遵循许多MoE 现有技术，我们还强制平衡和多样化的专家使用，避免崩溃。

...



### 融合跨场景一致性和空间平滑度到MoE

架构级别：共享永久专家

损失级别：编码几何感知平滑度的空间一致性损失



#### 共享永久专家

![image-20230825150026664](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/dc2f6e63df2c6baaf3b85845a7b62dac--ae29--image-20230825150026664.png)

跨损失的不一致



泛化：MoE应该对来自不同场景的相似外观模式或相似材料保持一致的专家选择

最终输出

![image-20230825150239888](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/f75cdf60963c39d4406e9f02f0610c29--42ec--image-20230825150239888.png)



#### 几何感知的空间一致性

给定空间上接近的两个点xi 和xj，路由器将其token embedding作为输入，分别映射到专家选择分数 R(xi)、R(xj)，拉近这两个分布，促使相似的专家选择

但计算所有点的成对距离太复杂



根据光线在图像坐标系中的位置计算**光线之间的成对距离**

过滤出距离小于预定义阈值ε的近距离光线

对于从两条近距离射线采样的点，计算所有点的欧几里德距离d

对于每个点 xi，我们选择其最近的点 xi'，距离为di,i′

通过对称KL散度损失，来鼓励最近点之间专家选择的一致性

![image-20230825150626296](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/5651da34575fe0996a3bd6e053e66e14--3685--image-20230825150626296.png)



更接近的点可能具有更高的专家选择相似度，因此我们不会平等对待所有的配对。

相反，我们使用它们的几何距离作为一致性置信度![image-20230825151422490](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/8528e0cafb3d9d3c386c28ba73856fe2--e4e1--image-20230825151422490.png)



最终的空间一致性损失定义为

![image-20230825150648455](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-25/880613cc50ca0d79922e6f3c925380bc--f477--image-20230825150648455.png)



空间一致性是在多视图的3D点上强制执行，自然地鼓励同一场景中几何感知的空间平滑度