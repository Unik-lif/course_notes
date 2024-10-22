## Lec0: Search
### Search
agent: 一个根据环境输入决定在当前环境做何种事情的对象

state: 对于agent以及环境情况的总体描述

initial state: 最早时候的状态

goal: 最终我们想要达成的目标

result(s, a): 表示根据在 state s 下执行了操作 a 之后最后返回的状态

state space: 全部我们所能够获得的状态集合

goal test: 决定是否某个状态和我们的目标符合

path cost function: 以防止代价太大

### solving search problem
- start with a frontier that contains the initial state
- repeat:
- - frontier empty, no solution
- - remove a node from the frontier
- - if node contains goal state, return the solution
- - expand node, add resulting nodes to the frontier

revised:
- start with a frontier that contains the initial state
- start with an empty explored set
- repeat:
- - if the frontier is empty, then no solution
- - remove a node from the frontier
- - if node contains goal state, return the solution
- - add the node to the explored set
- - expand node, add resulting nodes to the frontier if they aren't already in the frontier or the explored set

use stack: last-in first-out data type. => depth first search

use first-in first-out data type => breadth first search 

BFS & DFS: uninformed search, not care about the structure, don't use problem-specific knowledge to find solutions.

informed search: use problem-specific knowledge to find solutions more efficently.
### Greedy best-first search
One of the informed search.

对于这种类型的路径搜索，可能先找一个中转点。每一步都希望我们能够离我们的目标更加的近。

不容易搞到全局最优解，可以优化
### A* search
$g(n) + h(n)$

g(n): cost to reach node n.

h(n): estimated length to get to the goal from n.

A* 在下面的情况将会是最优解：
- h(n)并不会出现错估
- h(n)是连续的，对于任何自己后续的点n'，开销h(n) <= h(n') + c,c是为了步数开销

### MiniMax
让计算机能够理解获胜和失败的意思，用数值来表示定义。

给定一个状态s,攻防双方一个为 MAX 方，一个为 MIN 方
- MAX picks action a in actions(s) that produces highest value of Min-Value(result(s, a)) [considering what the min players do]
- Min 则是相反的，类似的，方法论听起来是一样的

```
function Min-Value(state):
    if Terminal(state):
        return Utility(state)
    v = +∞
    for action in Actions(state):
        v = Min(v, Max-Value(Result(state, action)))
    return v
```
### Optimizations: Alpha-beta pruning
当我已经知道可以拿到一个最优的解，我就不必再去遍历了。

## Lec1: Knowledge
我们应该如何让计算机能够推理？
### Proposition
用字母来表述某一条语句

对于 P->Q ，当 P 为 false，我们就不关心 P->Q 了，因此 P->Q 在此时都为正。

<-> 则是 if and only if

model: 根据语句输入判别正确或者错误

Entailment: 可以视作后面的语句在当前语句的 entail 下一定是在前面语句为真的情况下为真的

knowledge base: 多条语句可以构成知识库

Inference: 根据 knowledge base 所做的合情推理
#### Inference Algorithm
Model checking:
- To determine if $ KB \models \alpha $: 遍历所有可能的 model 情况，如果 KB 为 True 时 $\alpha$ 也一直为 True,那么就通过检查。
- KB 就是 Knowledge Base
- $\alpha$ 是我们的 Query 用于询问的问题 

通过我们的知识库来对机器进行提问，机器通过谓词逻辑来决定该提问是否正确。

这边记得看一下代码哈哈，我觉得真的非常有意思。

这件事情也被称为Knowledge Engineering.

该机器的好处是：可以解决一些Logic Puzzles,而这些谜题往往不是那么轻易就能figure out。

特别的，KB并不是一般的逻辑，而往往是逻辑的整体，是给定的条件，因此不要把它当正常的逻辑词来处理，而把它当做已知信息。
#### 一些公理
- 德摩根
- implication reduction
- bicondition
- etc.

#### First-Order Logic
- Person(Minerva): Minerva is a person
- House(gryffindor): Gryffindor is a house

This give us ways to express our logic better.

## Lec2: Uncertainty
Bayes' Rule
$$P(b|a) = \frac{P(a|b)P(b)}{P(a)}$$

Knowing P(medical test result| disease), we can calculate P(disease | medical test result)

Bayes' network

> 联系的普遍性

将某件事情作为Node,对于多件事情，利用edges来进行链接，然后合并在一起来考虑某件事情可能发生的概率。

主要是确认其他条件为恒定，检查当前我们需要再添加的额外条件所对应的概率。

### Inference
Query X: variable for which to compute distribution

Evidence variables E: observed variables for event e

Hidden variables Y: non-evidence, non-query variable

Goal: Calculate P(X | e)

Enumeration:

$$P(X|e) = \alpha P(X, e) = \alpha \sum _{y}{P(X, e, y)}$$

### Markov assumption
the assumption that the current state depends on only a finite fixed number of previous states.

The Markov chain is based on Transition Model.

AI 学习的一般是 Observation，机器不易学习的东西则是 Hidden State ，这两个之间往往存在很多联系。

比如伞和天气在下雨。这边的 Markov 链同样也可以用 Hidden 的来做

马尔科夫链因此存在针对hidden状态的链条，比如通过雨伞携带情况，来推断下雨的情况。这并不会直接相关，但是会存在较大关联。雨伞携带情况即为evidence，而天气则是hidden state。
## Lec 3:
优化问题，考虑爬山模型，因为局部最优解的存在，存在一些爬山找邻居的策略，以更可能找到全局最优解。

### Simulated Annealing
从一个高温系统，慢慢稳定下来，得到一个解决方案。

在一开始提高更大的随机性，在之后降低随机性。

一开始提供一个更高的`Temperature`，之后，则降低`Temperature`

通过添加一个概率修正，以让我们存在一定概率挑选更差的邻居值，以方便找到最后的全局最优解。

旅行商问题：NP complete问题

没有一个已知的很快的方法去找到解决方案

但是可以找到相对好的解
### 限制与约束图
基本上限确认目标，然后再确认约束条件，之后开始优化训练。

Arc consistency:
- 两个节点之间建立联系，要删掉其中一个节点所有会与Y相矛盾的情况可能，一直这么做直到稳定下来。 

AC-3算法，特别需要要重新把Z放进来，因为X的情况发生了更新。

backtracking search
- 一个寻找稳态的递归类型算法，有点像深度优先搜索
- 但是代价有点大，有很大的计算开销
- 添加推理来减少计算分支数目
- 可以避免多次的backtracking次数

maintaining arc-consistency
- enforcing arc-consistency every time we make a new assignment

每次我们为某个变量添加新的值，就跑一次AC-3算法

其他减小搜索空间的方法：
- MRV：找剩下值情况最少的变量
- start from the one with highest degree：触手更多
- 挑一个选项，能够尽可能地去排除其他选项

感觉就和做数独的逻辑是一样的，好神奇。

## Lec 4:
### Supervised Learning
Human tell the machine the result.
### Nearest neighbor classification
given an input, chooses the class of the nearest data point to that input

k-nearest-nighbor classification: A better way to choose the most common class, using k nearest data points to that input.

### Perceptron Learning
Using one single line to draw conclusion

$ w_0 + w_1x_1 + w_2x_2 \geq 0 \, return \, 1 \, else \, return \,  0$

If we choose weight vector $w: (w_0, w_1, w_2)$, input vector: $(x_0,  x_1, x_2)$, then we can use the dot product.

#### Learning rule
given data point $(x, y)$, update each weight according to:
$w_i = w_i + \alpha(y-h_w(x))  \times x_i$

where $y$ is the actual value, $h_w(x)$ is the prediction.

#### Hard threshold vs Soft threshold
Hard: we don't have a feeling like we are less or more sure about sth.

Soft: a logestic-like curve.
### Support Vector Machine
Many types to seperate the two categories well
### regression
supervised learning task of learning a function mapping an input point to a continuous value

Try to draw a line and find an estimate function to get the results based on the input.

### How to evaluate
Use a Loss function, take into account => How far it is for $| actual - predicted|$ => L1 Loss

L2 Loss: $(actual - predicted)^{2}$

### Overfitting
a model that fits too closely, lose great generality.

TO avoid, do some modifications:
$cost(h) = loss(h) + \lambda complexity(h)$

also measure the complexity, which will kinda avoid the overfit phenonmenon
### holdout cross-validation
splitting data into a training set and a test set, such that learning happens on the training set and is evaluated on the test set.

k-fold cross-validation:
- splitting data into k sets, experimenting k times, using each set as a test set once, and using remaining data as training data.
### Scikit Learn
A great way to accelerate your speed to use models.

### Reinforcement learning
given a set of rewards or punishments, learn what to do and what not to do.

### Markove Decision Process
model for decision-making.
- set of states $S$
- set of actions $ACTIONS(s)$
- transition model $P(s'|s, a)$
### Q-Learning
Learning a function $Q(s, a)$, estimate of the value of performing action $a$ in state $s$

Based on the result, the machine will learn from.
- start with $Q(s, a) = 0$, for all $s$,$a$
- when we taken an action and receive a reward
- - estimate the value of $Q(s, a)$ based on current reward and expected future rewards.
- - update $Q(s, a)$, take int oaccount old estimate as well as the newer version of estimate


$Q(s, a) \leftarrow Q(s, a) + \alpha (new \, value \, estimate - old \, value \, estimate)$

If $\alpha$ is one, this will be a stateless loop. $\alpha $ controls how important the new information is. 

new value estimate: the reward I get, and the reward of future.

$Q(s, a) \leftarrow Q(s, a) + \alpha ((r + \gamma max_{a^{'}} Q(s', a')) - Q(s, a))$

reinforcement learning:
- 如果AI知道某个路是对的，会形成路径依赖，但可能有更好的其他路，就是懒得探索
- EXPLORE vs EXPLOIT 的问题

$\epsilon$-greedy:
- with $1 - \epsilon$, choose the estimated best move
- with $\epsilon$, choose a random move
### Unsupervised learning
given input data without any additional feedback, learn patterns.

Clusterring.

Recentering until we converge:
- the mean point is steady
- no more points change their clusters

## Lec 5:
artificaial neural networks: based on the structure and parameters of the network.

using 'units' as neural 'nodes'.

$g(\sum^{n}_{0} w_i x_i + w_0)$

### gradient descent
Algorithm for minimizing the loss to train the neural network.
- start with a random choice of weights
- repeat:
- - calculate the gradient based on **all data points**: direction that will lead to decreasing loss
- - update weights according to the gradient

### Multi-layer NN
with an input layer, an output layer, and at least one hideen layer.

backpropagation: 
- start with a random choice of weights
- repeat:
- - calculate error for output layer
- - for each layer, starting with output layer, and moving inwards towards ealiest hidden layer
- - - propagate error back one layer
- - - update weights

To minimize the total loss.
### Avoid overfitting
dropout: randomly remove units, select at random, make the system more robust.

当神经网络的层数变多之后，一般来说它可以处理更加复杂的任务，课程中使用了classifier来做示例。

### computer vision
Great amount of computations: how to make it practical.

image convolution: filter and extract, take a pixel based on neighbors.

比如某些filter可以在计算时，让图像的边界变得更加明显，能够明显区分两个不同的色块，从而在目标检测上有一些优势。

Pooling: reducing the size of an input bt sampling from regions in the input.

max pooling: choose the maximum value in each region.

convolutional NN: CNN
- using convolution in NN. Before NN, using convulution methods first.

recurrent NN: fed back into itself

通过多次交互和反馈来逐渐贴近我们的需求
## Lec 6:
Formal Grammer is important for NLP, which is a system of rules for generating sentences in a language.

First, define a context-free grammer. 使用ntlk，但是这样需要研究很多语言，这是不经济的。

n-gram: a contiguous sequence of n items.

Feeling and Sense: Naive Bayes.

additive Laplace smoothing: add one to each value in our distribution to avoid 0 cases.

后面的word2vec的知识，似乎是在做LLM Sidechannel的时候学习了的。

有趣的地方：
```
closest((king - man) + woman) = queen
```
这种东西很适合做翻译，毕竟它更加注重词与词之间的关联。

encoder and decoder: Encode the input to get the hidden state.

### Attention
decide which value and hidden state counts more.

When decoding, which input should be paid more attentions to?
### Transformers
process each input word in parallel and independently.

Add the positional encoding, therefore it captures more info.

Add self-attention (always use multiple self-attentions to get other words info and context info, get info that is useful)

In decoder, we will add encoded representations for references to generate next output word.