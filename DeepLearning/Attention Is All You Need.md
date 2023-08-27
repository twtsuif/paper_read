# Attention Is All You Need



## 架构图

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-23/07de84bb0eaa1cb3f15ce6b984191ba1--746f--image-20230623230023540.png" alt="image-20230623230023540" style="zoom: 80%;" />

堆叠式自注意力和逐点，完全连接的encoder和decoder层



## 比较

![image-20230712114045575](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/e591f219947055a42c26f2f8bd8d05e3--fab9--image-20230712114045575.png)

不同层类型的最大路径长度、每层复杂度和最小顺序操作数。

n 是序列长度，d 是表示维度，k 是卷积核大小，r 是受限自注意力中邻域的大小。



## Abstract

主流的序列转换模型基于复杂的循环或卷积神经网络，包括编码器和解码器。

性能最好的模型还会通过一个注意力机制连接编码器和解码器。

我们提出Transformer，完全基于注意力机制，省去循环和卷积。

性能优越、更高并行性

2014英德翻译提高了2个BLEU 2014英法翻译单一模型最先进的得分

通过应用English constituency parsing，大量数据or有限数据，展示了Transformer可以推广到其他任务



## Introduction

无数的努力在推广着循环神经网络和编码器-解码器体系结构的范围

循环模型将位置与计算时间步对齐，生成隐状态序列ht

固有的序列性排除了训练示例的并行性，在序列长度较长时至关重要，内存约束限制了跨示例的批处理

最近的工作通过分解技巧和条件计算显著提高了计算效率，后者也提高了模型性能，但顺序计算的限制还在



注意力机制已经成为各种任务中，序列模型和转换模型的重要部分

允许对依赖关系进行建模，无需考虑在输入或输出序列中的距离

除了极少数情况，这种注意力机制都与循环网络结合使用



我们提出了Transformer，避免了循环，而是完全依赖于注意力机制来绘制输入和输出的全局依赖关系

允许更多的并行化

在8个P100上训练12个小时，达到最先进的翻译质量



## Background

减少顺序计算的目标形成了Extended Neural GPU、ByteNet和ConvS2S的基础

这些都使用卷积神经网络作为基础构建块，对所有输入输出位置并行计算隐层表示

这些模型中，将来自两个任意输入或输出位置，相关联所需的操作数随位置之间的距离增长，ConvS2S线性，ByteNet对数，学习远位置之间的依赖关系困难

Transformer中这被减少到一个恒定的操作次数，尽管代价是由于平均注意力权重位置降低了有效分辨率，这一效应我们用多头注意力抵消



自注意力，有时也称内部注意力(intra-attention)，将单个序列的不同位置联系起来，来计算该序列的表示，的一种注意力机制

自注意力已经用于多种任务，阅读理解、摘要总结、语篇蕴含、任务独立的句子表征的学习

端到端记忆网络基于循环注意力机制，而不是序列对齐的循环，并且已经证明在简单语言问题回答和语言建模任务中表现良好



Transformer是第一个完全依靠自注意力来计算输入输出的表示，而不使用序列对齐的RNN或卷积的转换模型

下面我们将描述Transformer、激活自注意力和讨论它的优势



## Model Architecture

encoder将符号表示的输入序列x映射到连续表示的序列z

给定z，decoder一次产生一个元素的符号输出序列y

每一步模型都是自回归的，生成下一步时，先前生成的符号作为额外输入



### Encoder

n=6个相同层的栈，每一层有两个子层：多头注意力机制和简单的位置全连接的前馈网络，两个子层中每一个子层使用残差连接，然后进行层的归一化

每一个子层的输出是![image-20230624153310100](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-24/96a97f59348f6a5a82d95b1c156bf82a--1d28--image-20230624153310100.png)

为了推动剩余连接，模型中的所有子层以及嵌入层产生维度dmodel=512的输出



### Decoder

n=6个相同层的栈，除了Encoder中的两个子层外，Decoder插入第三个子层：对Decoder栈的输出执行多头注意力

与Encoder类似，我们在每个子层周围使用剩余连接，然后进行层的规范化

修改了自注意力子层，避免位置关注后续的位置

这种掩蔽，结合输出嵌入偏移一个位置的事实，确保了对位置i的预测只能依赖于小于位置i的已知输出



### Attention

<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-06-24/ee247001c0917ebae4b40122b48be952--751c--image-20230624155216272.png" alt="image-20230624155216272" style="zoom:80%;" />

注意力函数：将q和一组(k,v)映射到输出，输出被计算为v的加权值，权重由q与k的兼容性函数计算



#### Scale Dot-Product Attention

![image-20230712105200884](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/5da7a97ecd865d7e5898155db84baa3e--5fe6--image-20230712105200884.png)

dk是q和k的维度

最常用的注意力是加性注意力和点积注意力，除了缩放因子根号dk外，点积注意力与我们算法相同

加性注意力，使用具有**单个隐藏层**的**前馈网络**来计算**兼容性函数**

复杂性在理论上相似，但点积注意力在实践中更快、更节省空间，因为可以使用矩阵乘法

虽然对于较小的dk值，两种机制表现相似，加性优于点积

我们怀疑较大的dk值，点积幅度变大，将softmax推入梯度较小的区域，为了抵消这种影响，我们开根号分之一



#### Multi-Head Attention

多头注意力允许模型共同关注来自不同位置的不同子空间的信息

![image-20230712110248055](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/363b2d862b92d22b3be602341fe09857--d563--image-20230712110248055.png)

我们采用 h = 8 个并行注意力头，每一个，我们使用 dk = dv = dmodel/h = 64。每个头的维度减少，总计算成本与全维度的单头注意力相似



#### Applications of Attention in our Model

Transformer使用三种不同的方式使用多头注意力

- encoder-decoder attention层中，q来自先前的decoder层，内存k和v来自编码器的输出，允许decoder每个位置都attend输入序列中的所有位置，模仿了seq2seq模型的典型encoder-decoder注意机制
- encoder包含自注意力层，编码器中每个位置可以关注编码器先前层的所有位置
- decoder类似，我们需要防止decoder的左向信息流来保证自回归，通过掩蔽softmax



### Position-wise Feed-Forward Networks

全连接的前馈网络，单独且相同地应用于每个位置，由两个线性变化组成，中间有一个ReLU激活

![image-20230712113038935](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/ff3cde3287f60fa631d8aa82bcf912ad--3f71--image-20230712113038935.png)

虽然不同位置的线性变化相同，但层与层之间使用不同的参数

另一种描述方式是内核大小为 1 的两个卷积，输入和输出的维度为 dmodel = 512，内层的维度为 dff = 2048



### Embeddings and Softmax

与其他序列转导模型类似，我们使用学习嵌入将输入标记和输出标记转换为维度 dmodel 的向量

还使用一般学习的线性变换和 softmax 函数将解码器输出转换为预测的下一个token概率

在我们的模型中，我们在两个嵌入层和 pre-softmax 线性变换之间共享相同的权重矩阵，类似于[30]。

在嵌入层中，我们将这些权重乘以 √dmodel。



### Positional Encoding

使用不同频率的正弦和余弦函数

![image-20230712113747956](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-07-12/1984441bfbd1a5e810e43aa95f876c62--393d--image-20230712113747956.png)

pos是位置，i是维度

位置编码的每个维度对应于正弦曲线。

波长形成从 2π 到 10000·2π 的几何级数。

我们选择这个函数是因为我们假设它可以让模型轻松学习关注相对位置，因为对于任何固定偏移 k，P Epos+k 可以表示为 P Epos 的线性函数

我们还尝试使用学习的位置嵌入，发现这两个版本产生几乎相同的结果。

我们选择正弦版本，因为它可以允许模型推断出比训练期间遇到的序列长度更长的序列长度
