# hw1:
## course 1: 机器学习与深度学习
让机器掌握寻找相关函数的能力。

机器学习有两大任务：`Classification`和`regression`，除了这两类任务以外的任务，一般是产生有结构的东西，需要让机器学会创造这件事情，这件事情存在难度，不过现在已经是`chatgpt`的时代。

机器学习经典三步骤
### 找一个模型：以linear model为例子
基本模型：
$$
y = w x + b
$$
其中`w`是`weight`，而`b`是`bias`。
### 找一下Loss与差距
定义训练数据的`loss`：
$$
L(b, w)
$$
在选用特定`b`与`w`时出现的`Loss`，这个`Loss`需要通过训练资料来得到。
### 优化，找到最小的Loss存在情况
假设有`N`个数据样本：
$$
L = \frac{1}{N} \sum _{n} {e_{n}}
$$

`e`即`error`，存在多种选用方式，面对不同类型的数据处理需求。

`Gradient Descent`来进行优化:
1. 首先我们随便找一个点，确定该点的`w0`
2. 计算这个点的微分值，即斜率
3. 利用斜率和`learning rate`，符号记为$\eta$，来向下计算新的`w0`，即`hyperparameters`。通过$\eta \frac{\partial L}{\partial w}$来更新我们的`w0`的值

新的`w`计算方式：
$$
w_1 = w_0 - \eta \frac{\partial L}{\partial w}
$$

两种类型：`local minima`以及`global minima`作为极小值，`gradient Descent`有`local minima`的问题

放到这边，就把`b`也一起拿来做。

当然上面的那件事情还没有完成机器学习，还需要做预测。

可能要需要对七天的观看人数都做训练，相对于考虑每天的情况，线性拟合的情况会更好，因为所采用的变量数目更多了。

## course 2: 其他情况
### Piecewise Linear Curves
通过多个线性来组成一些线性的，存在转折的线性曲线。不过这种类型是单一的变量类型。

只要取的粒度足够细，`piecewise linear curves`可以逼近任何一个曲线。

但是线性并不是一个比较好的函数，如下所示：
```      
Hard Sigmoid Function
         /------------------------
        /
-------/
```
可以用一个`sigmoid function`来对其做逼近。
$$
y = c \frac{1}{1 + e^{-(b + wx_1)}} = c * sigmoid(b+wx_1)
$$
需要使用各种各样的`b`与`w`来组成不同类型的曲线，这是一个简单的函数平移问题。

对于任意一条曲线，都可以用这个式子来做逼近：
$$
y = b + \sum_{i} c_{i} * sigmoid (b_{i} + w_{i} x_{i})
$$

现在如果我们选取多个`features`，即使用多种类型的变量，原本我们做线性逼近可能是这么写一条式子：
$$
y = b + \sum_{j}w_{j}x_{j}
$$
但我们可以得到的结果如下所示：
$$
y = b + \sum_{i} c_{i} * sigmoid (b_{i} + \sum_{j}w_{ij} x_{j})
$$
这边的`j`是`features number`，这边的`i`是`sigmoid number`，因此$w_ij$其实就是`weight for x_j for i-th sigmoid`。

那么最后这个函数，我们是以`sigmoid number`来进行观察，每个`sigmoid number`对应了`j`中所有类型的`features`。

$$
r_i = b_{i} + \sum_{j}w_{ij} x_{j}
$$
可以用一个大矩阵和向量乘法来进行表示。
$$
r = b + WX
$$

简洁表示，继续做：这边的$\sigma$是`sigmoid function`
$$
a = \sigma(r)
$$
最后得到式子：
$$
y = b + C^{T} \sigma(r)
$$

现在我们开始训练，我们所不知道的未知参数是`W`，外面的`b`，`C`, 里面的`b`，将他们进行拼接，成为一个$\theta$，统称为未知参数。

现在针对$\theta$向量来进行`gradient decend`，同样先选取一个随机的$\theta_{0}$，之后就逐渐向下进行迭代。和之前一样。

怎么做优化？从一个大$\theta$中取出一部分，作为`Batch`，来进行综合处理，不一定细化到每个小变量的层面，可以整体考虑。这种调用`batch`的方式被称为`epoch`。`Batchsize`也是一个`HyperParameter`。

对于`HardSigmoid`，其实也可以被视作两个`ReLU`的合成。
$$
c max (0, b+ wx_1)
$$
这样往下做的结果是：
$$
y = b + \sum_{2i} c_{i} max (0, b_{i} + \sum_{j} w_{ij}x_{j})
$$
取一个好名字，有很多的`sigmoid`，看起来像神经元向下传递，那就叫神经网络吧。一排`neuron`就是一层`layer`，要好几层的话，那就是深度学习。

如果搞的太深了，很有可能会出现`overfitting`问题