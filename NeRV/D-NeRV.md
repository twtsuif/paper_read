# D-NeRV

Towards Scalable Neural Representation for Diverse Videos

多样化视频的可扩展神经表示



GitHub：https://github.com/boheumd/D-NeRV



<img src="https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/7454710f54cfb09f970cf1fe3dc8d18c--976e--image-20230821084329443.png" alt="image-20230821084329443" style="zoom:80%;" />

控制每个视频剪辑的关键帧



![image-20230821092247641](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/9f73822656b04280cdcafc882474ffc1--02d6--image-20230821092247641.png)

与NeRV的比较



## 架构图

![image-20230821093734402](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/cc76cef65f16a92ac82997e307920812--5322--image-20230821093734402.png)





## 摘要

现有的基于INR的方法仅限于少数具有冗余内容的视频，为每个视频帧设计不同的模型

统一的模型对视频联合编码，而非分成多个子集单独建模		// 无法利用视频的长期冗余

- 从运动信息中解耦clip-specific的视觉内容
- 引入时间推理到隐式神经网络
- 使用面向任务的流作为中间输出，减少空间冗余

大幅超过NeRV和传统压缩

用作数据加载器时，动作识别任务的准确率比NeRV高3-10



## 简介

与基于学习的视频压缩方法相比，基于INR，更简单的训练管道，更快的解码速度

当前的NeRV设计，回报递减diminishing returns

内容和运动的信息建模的耦合，扩大了记住不同视频的难度



观察到，视觉内容(前景和背景)通常代表外观，运动信息代表语义结构(可在不同视频中共享)

将视频解耦为两部分：clip-specific的视觉内容、单独建模的运动信息



视频任务中，时间建模相当重要，我们不是独立地输出每个帧，而是，通过显式建模不同帧之间的全局时间依赖性，时间推理引入到INR网络



考虑空间冗余，建议预测面向任务的流作为中间输出，与关键帧结合使用获得最终输出，减轻不同帧上记忆相同像素值的复杂性



## 相关工作

### 隐式神经表示

E-NeRV将隐式神经表示分解为单独的空间和时间上下文

NRFF和IPF预测连续视频帧之间的运动补偿和残差

CNeRV提出**具有内容自适应嵌入**的混合视频神经表示，进一步引入内部泛化



## 方法

引入视觉内容编码器，来编码**来自采样的关键帧**中的内容



### 视觉内容编码器

编码器E，捕获clip-specific的内容

与通过模型记忆不同的内容相比，我们还是建议关键帧

每个视频分为连续的clip，每一个都采样开始和结束帧(I0,I1)，多阶段提取视频内容 ![image-20230821095539376](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/a0d6390e495de7111b8b72d566bce1c9--1fe4--image-20230821095539376.png)

E由堆叠的卷积层组成，逐渐对关键帧下采样



### 运动感知解码器

利用关键帧的视觉内容，提供运动信息来重建完整视频

输入时间坐标和内容特征图

预测面向任务的流作为中间输出，**打包**生成的内容特征

提出了**空间自适应融合模块**，更有效地融合内容信息

通过提出的**全局时间MLP模块**，为解码器配备时间建模能力



##### 多尺度流量估计

预测每个timestep面向任务的流

第一阶段，两个内容特征图$I_0^1$和$I_1^1$，沿时间轴应用线性插值，生成每个中间时间步的特征图  ![image-20230821101248307](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/563e206cd4c96420544a8e7633244bc1--60ca--image-20230821101248307.png)

引入位置编码，与特征图$I_t^1$连接，第一阶段送到流估计G  ![image-20230821101520447](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/17bef736e34bd07588ae706cf4c00aca--8ec2--image-20230821101520447.png)

G是计算每像素流的卷积栈，在估计流![image-20230821101833312](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/6d79ed975778c701ee29c439eed9b617--0b69--image-20230821101833312.png)指导下，将关键帧的视觉内容传播到当前帧索引t



首先，通过双线性打包操作T，在**给定关键帧内容以及相应流**的时间索引t处，生成打包的前向或后向特征图![](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/ad561b866729d1c4e513afe65e39024c--6bba--ad561b866729d1c4e513afe65e39024c--a8d1--image-20230821102143617.png)

![image-20230821102753341](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/5171444b6a6c1a1fee4b1d04d62a297a--70de--image-20230821102753341.png)

为了以可靠方式融合前向和后向的打包特征图，设计一个距离感知置信分数，来对打包的特征加权求和，生成打包的特征 ![image-20230821103056205](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/e830eab890943a21296cd681ce833874--ff39--image-20230821103056205.png)



##### 空间自适应融合

融合clip-specific内容信息

motivated by modulations layers

特征传递到两个全连接层，学习逐像素的调制参数 ![image-20230821103807117](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/11635586b122fb8375762ed657b283aa--491b--image-20230821103807117.png)  ![image-20230821103850566](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/5fd3f91b5bce03666292c3743533d40c--c2d6--image-20230821103850566.png)

融合![image-20230821103922676](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/11c8c21942827853e357074340f194fd--d6e8--image-20230821103922676.png)

引入了由特征图指导的额外的归纳偏置，简单集成两个特征图

调制操作后，使用与NeRV相同的块架构，卷积层，GELU激活层，PixelShuffle层，逐渐对特征图上采样 ![image-20230821104636997](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/9ce2984444342527488b31e5c013f6a2--58f4--image-20230821104636997.png)



##### 全局时间MLP

受Transformer和基于MLP的模型在图像和视频识别的成功启发，引入MLP利用视频的时间关系

输入T帧的特征图$O^l$，将权重W应用于沿时间轴的每个通道，对全局时间依赖建模，残差相加 ![image-20230821105315218](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/c6c36dbbbbfa9b0ebf1134ca1859b41c--e907--image-20230821105315218.png)



##### 最后阶段

在时间t处，生成最后的重构帧$I_t^{'}$，连接解码器特征图$M_t^L$和打包的帧$\hat{I_t}$ 作为输入，输入两个卷积层进行最终细化



### 训练

重建帧和真值之间L1和SSIM损失的组合，进行与NeRV相同的优化，无需对流量估计进行任何显式监督

![image-20230821110303846](https://cdn.jsdelivr.net/gh/twtsuif/picture/twtsuif2023-08-21/6cc60489c740ac5bc08dfa0aa7c1f889--b897--image-20230821110303846.png)

数据集小批量，投入连续的视频剪辑，使用现有的图像压缩算法编码关键帧



## 实验

### 设置

#### 数据集

视频动作识别UCF101：101个动作13320个视频。以2fps 256x320提取所有视频，



标准视频压缩UVG   

视频修复DAVIS
