+++
title = "ZSL & FSL"
author = ["Linyu Yang"]
date = 2023-11-20T00:00:00+08:00
draft = false
math = true
+++

## Zero-Shot Learning {#zero-shot-learning}


### 简介 {#简介}

ZSL: 对从没见过的类别进行分类，使机器具有推理能力。
对于 <span class="underline">要分类的对象</span> 一次也不学习。具有吸引力。

```text
马，老虎，熊猫 -> 斑马（没见过）
```


### 数据集 {#数据集}

-   训练集数据 \\( X\_{tr} \\) ，训练集标签 \\( Y\_{tr} \\)
-   测试集数据 \\( X\_{te} \\) ，测试集标签 \\( Y\_{te} \\)
-   训练集类别描述 \\( A\_{tr} \\)
-   测试集类别描述 \\( A\_{te} \\)


### 类别描述 {#类别描述}

对于每一个类别 \\( y \in Y \\) 表示成语义向量 \\( a\_i \in A \\) 的形式。
语义向量：每一维度表示高级属性。

语义向量举例：

```text
黑白色
有条纹
马的形状
```

希望用 \\( X\_{tr} \\) 和 \\( Y\_{tr} \\) 来训练模型，使之具有识别 \\( X\_{te} \\) 的能力。
所以需要类别的描述 \\( A\_{tr} \\) 和 \\( A\_{te} \\) 。

类别描述的获取

-   人工专家定义（效果好）
-   <span class="underline">海量</span> 附加数据集进行学习


### 学习目标 {#学习目标}

类别和标注的关系，所以需要描述才能进行对于没有见过的类别的识别。


### 总结 {#总结}

利用训练集数据训练模型，使得模型能够对测试集的对象进行分类；
但是训练集类别和测试集类别之间没有交集；
期间需要借助 <span class="underline">类别的描述</span> ，来建立训练集和测试集之间的联系，从而使得模型有效。


### 应用到本项目存在的问题 {#应用到本项目存在的问题}

-   本项目是否属于分类问题？（参数生成、参数的编码）
-   如何获取合适的描述
    -   描述的编码
    -   使用 NLP 技术进行描述的获取（不现实）
-   建立什么的模型
    -   图像的分类问题如何与我们的问题产生关联


### 相关的文章 {#相关的文章}

-   zsl 的开山之作 (Lampert, Christoph H. and Nickisch, Hannes and Harmeling, Stefan, 2009)
-   zsl 的综述 (Fu, Yanwei and Xiang, Tao and Jiang, Yu-Gang and Xue, Xiangyang and Sigal, Leonid and Gong, Shaogang, 2017)
-   zsl 在和语言模型的应用
    -   大模型微调 (Wei, Jason and Bosma, Maarten and Zhao, Vincent Y. and Guu, Kelvin and Yu, Adams Wei and Lester, Brian and Du, Nan and Dai, Andrew M. and Le, Quoc V., 2022)
    -   自然语言分类器 (Srivastava, Shashank and Labutov, Igor and Mitchell, Tom, 2018)


## Few-Shot Learning {#few-shot-learning}


### 简介 {#简介}

FSL: 数据量小的分类 (小样本学习，人类擅长)
是一种 meta-learning (learning to learn) 。
（我们的项目是否适合 learn to learn）


### 数据集 {#数据集}

对于图像分类问题，需要训练集：大量的图像。

meta task

-   支撑集输入 \\( C \\) 个分类，每个类别有 \\( K \\) 个样本（support set）
-   从这 \\( C \\) 个分类中的剩余数据取一批样本作为预测对象（batch set），判断属于哪个类别

\\( C \\) - way \\( K \\) - shot

训练过程中，每次训练都会采样得到不同 meta-task，
所以总体来看，训练包含了不同的类别组合，这种机制使得模型学会不同 meta-task 中的共性部分，
比如

-   如何提取重要特征及比较样本相似
-   忘掉 meta-task 中 task 相关部分

通过这种学习机制学到的模型，在面对新的未见过的 meta-task 时，也能较好地进行分类。


### 学习目标 {#学习目标}

学习一个相似度函数，需要大量的数据集来实现这种学习


### 经典图像数据集 {#经典图像数据集}

Omniglot
Mini-ImageNet


### Siamese Network {#siamese-network}

1.  打标签，（ \\( x\_1 , x\_2 \\), 同类 ? 1 : 0 ）
2.  训练一个提取特征的神经网络
3.  计算相似度，sigmoid
4.  训练
5.  做 n-way one-shot 预测


### Triplet Loss {#triplet-loss}

1.  锚点
2.  正负样本，
3.  \\( Loss(x^a, x^+, x^-) = \max\\{ 0, d^+ + \alpha - d^- \\} \\)


### 应用到本项目存在的问题 {#应用到本项目存在的问题}

-   如何选取典型源代码文本（支撑集）
-   源码之间的相似度的学习，选取什么样的相似度函数
-   没有解决 <span class="underline">数据集标注不够</span> 的问题，因为仍然需要标注大量的数据进行相似度学习


### 相关的文章 {#相关的文章}

近年大模型和 few-shot 关联比较多

-   大模型的尺寸可以减小 (Schick, Timo and Schütze, Hinrich, 2021)
-   大模型微调技术 (Gao, Tianyu and Fisch, Adam and Chen, Danqi, 2021)
-   大模型与 few-shot (Perez, Ethan and Kiela, Douwe and Cho, Kyunghyun, 2021)
-   大模型重构代码 with few-shot (Shirafuji, Atsushi and Oda, Yusuke and Suzuki, Jun and Morishita, Makoto and Watanobe, Yutaka, 2023)


## 结论 {#结论}

-   语言模型主流的研究中，zsl 和 fsl 都是基于大模型的
-   任务效果未知（目标的特殊性、与现有工作的差异）
-   如果大模型微调技术可行，不应当把计算资源和时间浪费在尝试这两种方案上，因为
    -   zsl 对数据集的要求：
        -   标注的数量可能变少，但没有变少很多，因为类别和描述之间的关系需要学习
        -   结构更可能变复杂（需要描述）
    -   fsl 对数据集的要求：
        -   仍然需要大量带有标注的数据进行学习
    -   其余没有任何优势
