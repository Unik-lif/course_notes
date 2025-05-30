## Lec1:
Def: give computers the ability to learn without being explicitly programmed.

讲了一下监督学习，非监督学习的一些概念，相对轻松的一节课。

## Lec2:
For the **cost function**.
$J(\theta) = \frac{1}{2} \sum^{n}_{i = 1} (h_{\theta} (x^(i)) - y^(i))^{2}$

$\theta_j := \theta_j - \alpha \frac{\partial}{\partial \theta_j} J(\theta)$

Least Mean Squares:

$\frac{\partial}{\partial \theta_j} J(\theta) = (h_\theta (x) - y)x_j$

Gradient Descent: Batch vs Stochastic
- Full Dataset vs Only One Training Data (each step you pick randomly a data)

## Lec3:
Linear algebra review: Easy.
## Lec4:
Locally weighted regression

paramatric and Non-paramatric:
- Fit fixed set of parameters to data vs parameters you need to keep grows linearly

局部权重regression，是假设我们更加注重局部的重要性，因为我们希望在某个局部空间，更加贴合我们的数据，从而得到一个在局部较好的预测结果。

Fit $\theta$ to minimize:$\sum^m_{i=1} w^{i}(y^{i} - \theta^T x^{i})^2$

$w^{i}$ tells how much attention we should pay to different part of data.

A relative standard choice:

$$w^{i} = exp (-\frac{(x^{i}-x)^2}{2\tau^2})$$

当样本和当前训练相差得很大的时候，这个$w$能够把差异弄得更加平滑，这样做了可以让曲线是一个`moderate`的状态，从而避免过拟合。

如果使用了非权重的，我们得到的训练样本是确定性的，但是如果有权重，我们可能会为了调整权重与部分，从而出现被迫要存下来我们先前样本的情况，以便之后认真调整。

可是如果非权重，那就不需要存这么老些东西了。


likelihood: 我们当前的样本是固定的，但是我们需要计算当前我们给出某些参数搭建的某个模型，作用在这些样本上，尤其是从output角度，也就是$y$的角度去研究时存在的概率分布。

### Classification
这一部分的数学推导很精彩，我看得很爽

### Newton's Method
牛顿法本来是用来求零点的，现在可以用牛顿法来找极值。

这边提到了海森矩阵，查阅资料就可以用了，其实应该是数学分析中不算太难的东西。
## Lec 5
### perceptron
Learn it for historical reasons.

感觉利用了向量相加后的几何直觉，当引入新的变量X之后，需要对theta做新的调整。
### Exponential Family
$p(y;\eta) = b(y) exp(\eta^T T(y) - a(\eta))$
- $\eta$: natural parameter
- $T(y)$: sufficient statistic
- $a(\eta)$: log partition function

伯努利分布和高斯分布似乎都可以用这个来进行描述
### Properties
Exponential Family has some properties studied before.

e.g. Expectations, variance.

1. MLE expect $\eta$ => concave (国外对于凹凸的定义和我们相反，从上面还是从下面看)
2. $E[y;\eta] = \frac{\partial}{\partial \eta}a(\eta)$
3. $Var[y;\eta] = \frac{\partial^2}{\partial \eta^2}a(\eta)$
See in MIT 6.041
### GLM: generalized linear model
Real - Gaussian, Binary - Bernouli, Count - Poisson, R+ - Gamma

这一部分的Lecture我似乎没有听得特别懂，明天继续复习一次。

Assumptions:
- $y|x;\theta$ ~ Exponential Family($\eta$)
- $\eta = \theta^T x$
- Test Time: output $E[y|x;\theta]$ => $h_\theta (x) = E[y|x; \theta]$

理解起来可能就是：根据需要的任务，选择合适的线性模型，而这些线性模型的参数我们可以根据模型类型事先就确定了（根据统计意义）。

### Softmax Regression
首先利用多个分类器$\theta^Tx$来做，得到指标，之后做exp处理，再做`normalization`，可以视作一种`Softmax`的方式。

第二次去听lecture发现印度小哥讲的形象又生动，真好！

### Cross Entroy
似乎借用了交叉熵的定义来介绍某种gradient descent，与GLM应该是殊途同归的。