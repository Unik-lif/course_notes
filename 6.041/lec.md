虽然我不是特别想做 AI ，但是可能还是逃不过这一关。这是未来。

## Lec 1
样本空间：取样时不会超过样本空间的范畴。

Sequence of Events：多个事件之间互不干扰，相互独立，他们的概率可以作为相加。
## Lec 2
单个元素集的概率均为 0 ，因此一个区域的概率不可以视作很多个点概率的累加。

P(A|B): probability of A given that B occurred.

$$
P(A|B) = \frac{P(A \bigcap B)}{P(B)}
$$

$$
P(A \bigcup B | C) = P(A | C) + P(B|C)
$$

提供了一个雷达的例子，计算反向的条件概率，A先于B发生，但我们计算了在B会发生的情况时，A会发生的情况，这一点非常反我们的直觉。这似乎也说明为什么我们要给新冠做多次检测。

条件概率的反向：变成因果关系的模型，这便是贝叶斯模型。有一个果，计算某件事情作为原因所能出现的概率。

一个哲学问题

## Lec 3
核心内容：独立事件的定义
### Review
$P(B) = P(A) P(B | A) + P(A^c) P(B | A^c)$

Bayes rules:
$$ P(A_i | B) = \frac{P(A_i)P(B|A_i)}{P(B)}  = \frac{P(A_i \bigcap B)}{P(B)}$$ 
### 定义
考虑连续丢两次骰子，部分条件概率是完全独立的。

$$P(B|A) = P(A)$$

这个意味着，A的出现并没有为B的出现提供任何有价值的信息，这个可以推出下面的：

$$ P(A \bigcap B) = P(A) P(B) $$


特别注意，Seperate并不意味着独立无关，反而意味着他们相关

如果在空间区间中，处于Seperate状态的A和B占据一部分，如果A发生，则B不会发生

此外，如果两个事件是相交的，也不代表他们就不是独立无关的，恰恰很有可能是独立无关的

注意pairwise和mutual independence的区分，pairwise是两两无关，并不代表三个或者四个组合在一起依旧无关。  

### 趣味
最后给出了一个国王的父母有两个孩子，这个国王的兄妹存在女孩的概率是多少 这样的问题

这种没有明显假设的题目，需要仔细地做好假设，它本身可能并不构成一个题目。
- 假设这个家庭正常生娃
- 假设这个家庭生娃直到生出国王

## Lec 4
使用已知的概率来解决计数问题，主要用的特性是独立性

两种选取方式：先从M集合挑选出K个，再从K个进行排布；或者直接从M集合挑选K个进行排布。

因此：bionomial
$$\binom{n}{k} * k! = \frac{n!}{(n-k)!}$$

使用这边的算式，来定义下面的东西：$0! = 1$

如何计算$\sum_{k=0}^{n} \binom{n}{k}$?

Total number of subsets = $2 ^ n$

## Lec 5
我们也可以考虑讨论随机变量，这才是更加有意思的。

一个随机变量，往往是一个函数，或者一个程序。

PMF：概率权重

$$P_X(k) = P(X = k)$$

$$E[X] = \sum_x xP_X(x)$$

$$E[g(X)] = \sum_x g(x)P_X(x)$$

多多思考定义上的事情，问题才能迎刃而解

## Lec 6
把条件概率添加到我们的随机变量分配中去

$$p_{X|A}(x) = P_X(X = x | A)$$

因此，条件期望也就是上面的定义做一点加权：

$$E[X|A] = \sum_x xp_{X|A}(x)$$

Memoyless: 抛硬币行为实际上是没有记忆的，与实验的其他部分没有关联。这是一个gemometric PMF，与从第几个开始来算，我们加上任何条件做条件概率的概率分布，最后得到的结果是一样的

这边举出了一个gemometric example，同样是抛硬币，假设正面向上的概率是`p`
$A_1 : (X = 1), A_2 : (X > 1)$

$E[X] = P(X = 1)E[X|X=1]+P(X> 1)E[X|X>1]$

最后得到结果$E[X] = \frac{1}{p}$

与多元微积分一样，概率也存在多元的情况
$P_{X,Y}(x, y) = P(X =x, Y = y)$

## Lec 7
把原本的事情，采用新的符号来做表示。

$P_X(x) = \sum_{y,z} P_{X, Y, Z}(x, y, z)$

$P_{X, Y, Z}(x, y, z) = P_{X}(x)P_{Y|X}(y|x)P_{Z|X, Y}(z|x, y)$

fix x, and change the value of y and z.

可以用变量A或者其他来表示条件概率所表示的特殊条件

建立概念与定义上的直觉，有时候可能要比单纯证明还要重要

在这边提供了一个计算下面这件事情的做法：通过换元，找一个新的变量来做表示：

$E[X] = \sum^{n}_{k=0}k\binom{n}{k}p^{k}(1-p)^{n - k}$

我们假设我们提供一个变量
$$X_i=\left\{
\begin{aligned}
1, &success \_ in\_ trail\_i\\
0, &otherwise
\end{aligned}
\right.
$$

用这个变量来帮助我们求解，之后把上面的情况加起来，就能得到$E[X]$

## Lec 8
连续的概率：需要使用微积分来帮忙计算

$P(a \le X \le b) = \int_{a}^{b}f_X(x)dx$

引入了密度函数：probability density function

CDF: cumulative distribution function

$F_X(x) = P(X \le x) = \int_{-\infty}^{x}f_X(x)dx$

这是一种面对离散还是连续都通用的一个标识符。

Standard Normal $N(0, 1): f_X(x) = \frac{1}{\sqrt{2\pi}}e^{-\frac{x^2}{2}}$

$E[X] = 0, Var(X) = 1$

## Lec 9
今天听完`quiz 1`前最后的一节课，明天开始看书和做题。

Joint PDF

$P((X, Y) \in S) = \int\int_S f_X,Y(x,y)dxdy$

Needle of BUffon：布冯针

大体上这节课还是从原本的离散，转向这一节的连续，简单看一下就好。

## Lec 10
正常情况下，我们输入一个X，拥有`Px(x)`与`fX(x)`信息，通过条件概率，得到Y。

现在我们用贝叶斯，假设Y是先决条件。

$P_{X|Y}(X|y)$

replace P with f.

假设我们的检验存在噪音的干扰：
$Y= X + W$

Discrete X, continuous Y:
$$P_{X|Y}(x|y) = \frac{P_X(x)f_{Y|X}(y|x)}{f_Y(y)}$$

这一章用了链式法则，结合了求导时的一些事项，很值得学习。

## Lec 11
Covariance

$cov(X, Y) = E[(X - E[X])(Y - E[Y])]$

Correlation Coefficient

$\rho = E[\frac{(X - E[X])}{\sigma_X}\frac{(Y - E[Y])}{\sigma_Y}]$