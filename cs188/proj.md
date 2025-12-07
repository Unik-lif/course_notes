## Proj1
### Q1-Q4
前几个都算是课程的练手，对着伪代码来写就可以了。

可能需要注意的是A*会重新评估所有的结点到目标点的距离，因此即便已经遍历过了，也需要重新检查是否可能有更短的路径。为了达到这个目的，需要在扩展的时候，同步更新遍历的数据集合中的最短路程。

### Q5
利用一个新的list来记录当前路径是否已经访问了某个结点

特别的，需要注意要单独为每个fringe中的点维持一个新的list，因此deepcopy是必要的，否则路径将会彻底乱掉。

为了避免这个问题，我直接用位运算来做。

注意确实还是有很多容易搞错的细节，比如nextx和nexty不要和state搞混了
### Q6
最简单的方式其实就是找一个离自己曼哈顿距离最远的就好了

### Q7
写了一个丑陋的方法判断点在不同区域上的情况，分别讨论估算。如果在外头，则先到最近的点，再前后交叉左右判断

拿满分是没问题，但bonus拿不到
### Q8
直接用UniformCost来做就好了，它就是干这个的

## Proj2
### Q1
GPT太聪明了，做这个实验还是避免在代码仓库中使用它吧。

GPT考虑了我没有考虑的东西，比如当前的Ghost状态和与自身的距离。但实际上只考虑最近的minFoodDistance，似乎还是很快就能满足题目中的要求。
### Q2  
搞清楚每一层的深度表示的是什么，之后就是一个工程问题

### Q3
仔细理解清楚$\alpha, \beta$具体是做什么的，之后这件事情就不难理解了，多多演绎推理思考就好

### Q4
这个加起来直接算一下就好了

### Q5
可以考虑到ghost，dot，以及特效药的距离，选值需要仔细一点，不能太狂野，慢慢调试就行了

## Proj3
并不困难，关键是理解清楚，之后就只是巩固和复习。写码时有一些函数的作用可能不太清楚，理解可能也很难一下子到位，所以建议从最顶层的函数来写，像SICP中提示的那样，自顶向下地去做，否则确实很容易搞混。
### Q1
对着公式敲就好，注意iteration得用上，不然就搞错了
### Q2
蛮有意思的，不过多多尝试一下值就好。唯一需要注意noise看起来是确认的，我估计是算着期望来的，感觉也挺合理。

绕远路的方式需要noise设置的比较高
### Q3，Q4，Q5
本质上是工程问题，慢慢写就行
### Q6
注意所有的w是同时更新的，这一点很容易在写代码的时候遗漏掉，以及一开始的w是没有值的，需要自己赋值0然后再慢慢学习

其他的就很简单啦！

## Proj4
前三个问题搞清楚概念就好

### Q4
第四个问题很有难度，感觉主要是在于理解上

虽然就算术上我们很容易理解Variable Elimination，但是实际上程序写的时候是有一些技法
- 所有的evidence本质上都是条件变量，因此我们要维护Factors序列，让其能够一直得到使用
- 每次做elimination的时候存在仅删掉对应的variable，而真正与之无关的变量多个存在的情况，因此最后一步似乎还是要进行joinFactors的联立
- unconditioned variable在factor中如果只有一个，且对应要消除掉的hidden variable正好是它，那没有必要继续了，反正算起来也是1，在编程范式中似乎也把这一步乘1给略过了
- 参考一下lec中手动去做消元的过程就可以了，清除掉所有的hidden variable，也就是除了evidence Variable和unconditioned variable之后，我们得到了我们想要的，但是还需要进行最后一次join和normalize才能完成最终的规约化

真正上手做了才知道为什么这么搞。。。

### Q5
这个可能不是特别困难，搞清楚定义自己慢慢写就行
### Q6
感觉本质上也是定义游戏
### Q7
注意，这里的也是独立和Q6一样，搞一组用于判断的beliefs数据

因此，对于所有可能的oldGhostPos，都需要搞出来一个数据，这个数据将会用于预测下一个position的位置，和对应的期望值

以这个为基础，去尝试对接下来的位置做更新

和Q6的更新方式类似，其实是两种独立的，不同的beliefs更新方式

## Proj5
### Q1
确实没有使用过torch来真正做这样的工作，总体来说，有一些有意思的细节

torch用Parameter类型来表示需要进行gradient descent迭代的变量

no_grad表示在运行这个函数前，不要默认地去迭代变量，这样能够节省很多内存上的开销
```
Question q1
===========
*** q1) check_perceptron
Sanity checking perceptron...
/home/link/Desktop/UCB-CS188/machinelearning/autograder.py:333: DeprecationWarning: __array__ implementation doesn't accept a copy keyword, so passing copy=False failed. __array__ must implement 'dtype' and 'copy' keyword arguments. To learn more, see the migration guide https://numpy.org/devdocs/numpy_2_0_migration_guide.html#adapting-to-changes-in-the-copy-keyword
  expected_prediction = np.where(np.dot(point, p.get_weights().data.T) >= 0, 1, -1).item()
Sanity checking perceptron weight updates...
Sanity checking complete. Now training perceptron
/home/link/Desktop/UCB-CS188/machinelearning/autograder.py:393: DeprecationWarning: __array__ implementation doesn't accept a copy keyword, so passing copy=False failed. __array__ must implement 'dtype' and 'copy' keyword arguments. To learn more, see the migration guide https://numpy.org/devdocs/numpy_2_0_migration_guide.html#adapting-to-changes-in-the-copy-keyword
  accuracy = np.mean(np.where(np.dot(dataset.x, model.get_weights().data.T) >= 0.0, 1.0, -1.0) == dataset.y)
*** PASS: check_perceptron

### Question q1: 6/6 ###

Finished at 12:26:56

Provisional grades
==================
Question q1: 6/6
------------------
Total: 6/6
```
### Q2
这好像我第一次自己尝试去写一个神经网络，并且用pytorch去做迭代和训练

epoch跑了5000次就能达到很理想的结果，我们的结构也足够简单，说白了就是一层relu，两层线性，似乎这个东西可以利用Sequential来组织，不过我们这边在forward中把算式写好了，应该是一样的
```python
    def __init__(self):
        # Initialize your model parameters here
        # We first use one hidden layer.
        "*** YOUR CODE HERE ***"
        super().__init__()
        self.batch_size = 64
        self.lr = 0.001
        self.linear_layer1 = Linear(1, 300)
        self.linear_layer2 = Linear(300, 1)
    def forward(self, x):
        """
        Runs the model for a batch of examples.

        Inputs:
            x: a node with shape (batch_size x 1)
        Returns:
            A node with shape (batch_size x 1) containing predicted y-values
        """
        "*** YOUR CODE HERE ***"
        # pytorch will automatically adjust x, even when changing the size of the batch, the nn won't have to change.
        mediate = relu(self.linear_layer1(x))
        output = self.linear_layer2(mediate)
        
        return output
```
实验结果如下
```
(ml) link@public-Super-Server:~/Desktop/UCB-CS188/machinelearning$ python autograder.py -q q2 --no-graphics
/home/link/Desktop/UCB-CS188/machinelearning/autograder.py:282: SyntaxWarning: "is" with a literal. Did you mean "=="?
  assert all([(expected is '?' or actual == expected) for (actual, expected) in zip(node.detach().numpy().shape, expected_shape)]), (

Question q2
===========
*** q2) check_regression
Your final loss is: 0.000643
*** PASS: check_regression

### Question q2: 6/6 ###

Finished at 12:28:26

Provisional grades
==================
Question q2: 6/6
------------------
Total: 6/6

Your grades are NOT yet registered.  To register your grades, make sure
to follow your instructor's guidelines to receive credit on your project.
```
### Q3