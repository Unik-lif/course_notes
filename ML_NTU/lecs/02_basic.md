## 机器学习基本概念
让机器自动找一个函数去满足我们的需求。

根据函数输出来分类：
1. 回归：regression 函数的输出是一个数值
2. 分类：classification 函数的输出是一个类别（选择题）
3. 结构化学习：structured Learning aka generative Learning，生成式学习，是一个很难的话题

`chatgpt`是哪一类？

`chatgpt`实际上做的事情是接龙分类，利用概率做决策。但是，从使用者的角度来看，`chatgpt`说话说起来有很高的自由度，因此可以把它当做将生成式学习拆解成多个分类问题的机器。

### 函数的寻找
前置作业：决定找什么样子的函数，与技术无关

三步骤：
1. 设定范围：从候选函数的集合里找，Model（类神经网络的结构是一个候选函数的集合，CNN RNN Transformer Decision Tree之类的）
2. 设定标准：从集合中挑最好的，Loss越好，函数越好。需要让专业人士根据资料来做标注，这便是监督学习（Supervised Learning），`L(f)`的计算过程取决于训练资料。Semi-supervised Learning，则是有些没有标注，在这种情况下可能要做一些假设，当然假设不一定是对的。RL, etc.
3. 达成目标：找到最好的函数，最优化`Optimization`. Gradient Descent, Genetic Algorithm, etc.

`Loss`越小越好，把函数集合记作`H`，把`H`中的每个`f`带进去，来确定：
$$
f^{*} = arg min_{f \in H} L(f)
$$
需要区分不同面向的技术

### 三步骤怎么判断它是好的？
最佳化演算法，`Optimization Algorithm`在跑之前，需要先设定一些参数，`Hyperparameter`，即超参数，需要人手工句调整。特别的，好的演算法对超参数不敏感。

### 生成式学习两种策略：
1. 各个击破 -> autoregressive (AR) model，每个像素都需要等前面的像素生成，速度慢，品质高，常用于文字
2. 一次到位 -> Non-autoregressive (NAR) model，只要有足够的平行运算能力，就可以很快，速度块，品质不一定高，常用于影响

截长补短，在语音合成上是一个标准的解法。

先用各个击破的方式决定大方向，再用一次到位的方式来处理。

或者还有一种类似方式，多次使用一次到位的方式，让图片更加清楚精确。`Diffusion Model`。

### New Bing等能够使用工具的AI
`New Bing, WebGPT, ToolFormer.`

使用工具的`AI`，拥有一定的搜寻能力，并且何时进行搜寻是机器自己所决定的。不过也会有一些精确性问题，会乱讲。

与`New Bing`相关的很相像的工作`WebGPT`，令人好奇。

使用搜寻引擎，也是文字接龙。`langchain`，不停向下搜索直到迭代完毕，和人类搜索的过程也很像。似乎也是通过人类老师来做示范的，记录了人类搜索行为记录。

两招：
1. 左右互搏：另一个GPT来帮忙，得到数据
2. 验证语言模型生成的结果