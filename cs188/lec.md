## 一些标注
网上可能有很多版本，为了做练习方便，我们直接使用fa2018的版本。

## Lec1:
AI: computational rationality

Main goal: Designing Rational Agents

Agent: an entity that perceives and acts

rational agent: select actions that maximize its utility

percepts to the environment, and based on the action space, do some actions.

Done.
## Lec2:
### Agents
#### Reflex agents:
choose action based on current percept

may have memory of a model of the world's current state

But don't consider the future consequences of their actions

#### Planing Agents
Decision based on hypothesized consequences of actions.

Must have a model of how the world evolves in response to actions.**That is quite different from Reflex agents**.

### Search Problems:
consistutes:
- state space
- successor function
- start state and a goal test

Solution: a series of successor functions from start state to goal. 
### Iterative Deepening
Idea: get DFS's space advantage with BFS's time / shallow-solution advantages.
- Run a DFS with depth limit 1, if no solution ...
- Run a DFS with depth limit 2, if no solution ...
- Run a DFS with depth limit 3. ...

Don't have to store too much in your memory.

这边的解释有点抽象，但实际上就是给DFS限定范围，然后一层一层地向下找。
### Uniform Cost Search
顾名思义
## Lec3: A*
Search Heuristic: 启发式搜索

A function estimate how good this state is, how close the state is to the goal.

A*: combination of Greedy and UCS.

其实也就是计算现在的Cost，加上对于未来预期的Heuristic Cost，两者相加作为评价指标。

Only Stop when we dequeue a goal.

### Estimate Heruistic Cost
Maybe the Heruistic function is not so good.

A heuristic `h` is `admissible` (optimistic) if:
$$0 \le h(n) \le h^{*}(n)$$ 

h^{*}(n) is the true cost to a nearest goal.

我们的估计要比实际上的要更加乐观，要比真实的开销要小。

在admissible的情况下，A*算法是optimal的，可以证明任意的一个A的祖先结点会在B之前进入边缘。

利用这个admissible作为武器，我们可以很方便地解决本来可能非常棘手的问题。不过这边也会有一个权衡，通常启发式越好，则我们所需的工作量，即扩展的结点就会越少。

### Graph Search
我们不再把已经在路径里的结点重新添加进去了，用一个封闭的集合来做对已经经过的结点的存储工作。

但是，我们仍然存在走错路的可能性，尤其是当评估函数引导我们走入错路的情况时。

这就需要一致性，不仅要求有admissible，而且还要有代表性。代表性的本质上也就是f(A)小于等于f(C)，即f的值是一路上升的。
## Lec4: 
### Graph Search: Never expand a state twice.
Tree Search + set of expanded states ("closed set")

Important: Store the closed set as a set, not a list.

Some features:
- Admissibility: $h(A) \le$ actual cost $h^{*}$ from A to G => $A^{*}$ tree search is optimal
- Consistency: $heuristic "arc" cost \le actual cost for each arc$ => $A^{*}$ graph search is optimal

$$h(A)-h(C) \le cost (A to C)$$

一致性的本质是：只要它说这个方向是对的，那么它的大方向就一定是对的，不会出现大的浮动。就像是指南针，保证自己不会指向北方一样。

### Local Search
find configuration satisfying constraints, or find optimal configuration.

In such cases, can use iterative improvement. Keep a single "current" state, and then try to improve it.

Hill climbing:
- Simple, general idea: move to the best neighboring state

Eight Queen Problems.

### Simulated Annealing
Learned from CS50-Harvard.

### Beam Search
Basic Idea:
- K copies of a local search algorithm, initialized randomly

The searches communicate! That is the best part of this algorithm. => genetic algorithms.

## Lec4: CSPs
The goal itself is important, not the path.

### CSPs:
Constraint Satisfaction Problems
- standard search problems: State is a black box.
- a special subset of search problems. Goal test is a set of constraints specifying allowable combinations of values for subsets of variables.

Initial State: Empty, Goal State: all variables have its values.

Goal test is a set of constraints specifying allowable combinations of values for subsets of variables.

Key components:
- Variables
- Domains
- Constraints
- Solution
### Backtracking Search
basic uninformed algorithm for solving CSPs.

continuing until some faults happen.
- Ordering: which variable should be assigned next, and in what order should its values be tried?
- In what order should its values be tried?

Filtering:
- keep track of domains for unassigned variables and cross off bad options
- cross off values violating constraint when added to the existing assignment

an Arc X-> Y is consistent iff for every x in the tail there is some y in the head which could be assigned without violating a constraint

如果做不到iff，就得删除相关的元素

### AC3
似乎非常简单，说白了就是把所有的constraints赛进来，逐个条目进行检查。

### MRV: minimum remaining values
choose the variable with the fewest legal left values in its domain

## Lec5: CSPs II
forwarding checking 和 arc-consistency 的区别
- 前者直到出现错误结束
- 后者是发现未来某些情况不可信就去除，更加未雨绸缪

K-consistency
- any consistent assignment to k-1 can be extended to the k_th node
- expensive

3-consistency

### Tree CSPS
compared to general CSPs, where worst-case time is O(d^n), tree is much better.

can be solved in O(nd^2)

还是比较容易推理得到的

之后还引入了做割集的方式，

## Lec6:
Adversarial Search (minimax)
- for zero-sum games
- compute each node's minimax value

预测对手的行径，对手是希望你的分数变得更加的低，所以自己在做的时候，选择让对手让你最低分数最高的那一个分支
- 也就是假设自己的对手很聪明，能够拿到最优解
- 空间复杂度可能是O(bm)，这种情况就是某一条很长的路线往下摸索，直到考虑最后一步对手可能的决策

minimax pruning
- 去除掉不需要计量的部分，比如可能表现更小的分支，因为maxer不会让控制流进入到那一束

这边的函数定义还是有点奇怪的
```python
def min-value(state, a, b)
    initialize v = + maximum
    for each successor of state:
        v = min(v, value(successor, a, b))
        if v <= a return v
        # b需要被更新，a表示到state这个节点的最大值
        # 你的对手一定会找这个最大值
        b = min(b, v)
    # 最后返回一个子节点
    # 但似乎是告诉上面的结点，情况会比V更糟，当然可能不会访问v这一条
    return v
```

alpha-betra pruning其实自己去做还是挺麻烦的，需要通过练习来进一步地巩固

Depth matters

## Lec7:
Worst-Case vs Average Case

expectimax search: values should now reflect average-case (expectimax) outcomes, not worst-case (minimax) outcomes

主要是为了考虑真实情况下，你的对手也没有办法做到最优，比如丢硬币

唯一的问题是：没有办法pruning，因为所有的值都很重要

Expectimax Pacman: Optimistic thinking it is safe

Minimax Pacman: Pessimistic thinking it is dangerous

使用不同的策略确实会影响到最后的表现，尤其是在你的对手也可以调整攻击策略的时候

### Utility
Minimax might not be a good idea
- 所谓杞人忧天

Preferences:
- An agent must have preferences among prizes A, B, etc.

这里借用了数学分析里头的偏序的概念来展现 rational preferences 这一概念

一般化后其实就是MEU Principle，即Maximum expected utility principle

但是人性要比这个还要复杂一些，比如如果你能稳赢，但是期望更低，可是你还是会考虑稳赢

## Lec8:
Grid World:
- Noisy Movement: actions do not always go as planned
- Rewards: receives rewards each time step
- Goal: maximize the reward

In stochastic grid world, the outcome is underterministic

Markov Decision Processes
- A set of states
- A set of actions
- A transition function: probability that a from s leads to s'
- A reward function
- A start state
- Maybe a terminal state

MDPs are non-deterministic search problems
- Markov means action outcomes depend only on the current state
- use policy not a plan, a policy is a function to recommend your next action

Discounting
- Reward will decay by the time
- Prefer now to later

Bellman Function

Key Idea: your state is determined by the former states you've been gone through

## Lec 9
Convergence will happen, because reward decays at $\gamma ^{k}$ for depth of $k$, while $\gamma$ is smaller than 1, bigger than 1.

Optimal policy is correct but so slow, an alternative approach should be proposed:
- step1: policy evaluation
- step2: policy improvment

policy-extraction

Bellman Function

Key Idea: your state is determined by the former states you've been gone through

本质上是一个逆向递归的机制，用来让我们的估计可以变得更加精确。这边的former并非是父亲结点，而是上一时刻的状态值。
## Lec 10
RL: Reinforcement Learning
- Agent receive feedback in the form of rewards
- Agent's utility is defined by the reward function
- Must act so as to maximize expected rewards
- All learning is based on the feedbacks

problem is the same as MDPs, but the policy is vague. Learn from your experience.

Offline: MDPs vs. Online (RL)

The actual space is given, but we don't know the T and R.
- T: transition
- R: reward

Model-based and Model-free 
- based on an estimation model
- based on your own exact direct trails
- Model-based: weighting by probability
- Model-free: no probability, the random selection of samples have already reflected the probability

基本思路很简单，根据真实情况，确认Learned Model中的T和R值的分布情况，分别表示状态转移的可能性和对应的奖励

### Two types
#### Passive RL
Direct Evaluation: 直接观察Episodes变化，对不同状态赋值，但是忽略了本身不同状态之间值的转换

Sample-based Ealuation: Average the Experience

Exponential Moving Average: 某种数列
#### Active Reinforcement Learning
Use Q value => Learn the optimal policy/values

与Value Iteration相比，你不需要知道max对应的action，在开放型问题中很有用
### Costly
It will be costly, because you'll need infinite time to converge.

- use Simplified Bellman updates calculate V for a fixed policy

## Lec 11
引入一个$\epsilon$作为随机选择的可能性，用来在一开始提高coverage，但是之后则可以逐步降低这个值

Exploration functions, takes a value estimate u and a visit count n, and returns an optimistic utility
- e.g. $f(u, n) = u + k/n$

favoring actions we haven't gone through

Q value will propagate，这也意味着哪怕你对某个区域已经有了一定探索，探索所带来的奖励并不多时，依旧会有进入某个区域的惯性

### Regret
You wish you haven't done something you did.

Random epsilon learner will have much more regret than exploration function-based learner

### Approximate Q-Learning
In realistic Cases, Q-Values are too many to visit them all in training

How to learn small number traning states from experience, and then generalize that experience to new, similar situations
- combine features together: Linear Value functions
- Advantage: our experience is summed up in a few powerful numbers
- Disadvantage: states may share features but actually be very different in value

$Q(s, a) = w_{1}f_{1}(s, a) + \cdots + w_{n}f_{n}(s, a)$

如果这些feature本身比较少，强化学习的速度将会是飞快的。

本质上是机器学习中的概念，到强化学习领域的迁移，这是一个很自然的，老师的用词是，well-justified，说白了就是gradient descent的其中一种形式，很期待

但问题是，这本质上只是related，对于不同的特征走gradient descent是会有一定的效果，但是本身只能正相关反映，可能还是存在粗放狂野的可能性
### Policy Search
just try policy and find what is the best

这个方法相对于上个方法更加与我们的任务直接关联

## Sum
First Part: Search and Planning

Next Part: Uncertainty and Learning!!

## Lec 12
Skip, Trivial

Bayes' Rule: from known to unknown, backward inference => sounds into words and sentences.

## Lec 13
Bayes' Nets
- a set of nodes, one per variable x
- a directed, acyclic graph
- a conditional distribution for each node

A Bayes net = topology (graph) + Local Conditional Probabilities

A bayes' nets implicitly encode joint distributions
- $P(x_1, x_2, \cdots, x_n) = \prod_{i = 1}^{n}P(x_i | parents(X_i))$
- **key points**: independent parent $X_i$

需要注意的是拓扑结构的排序总是可以存在，因此总是能够构成一个链式结构，满足拓扑结构的需求。

贝叶斯网络的关键在于，我们首先要为变量选择一个与网络结构一致的排序。这意味着如果在有向图中存在从$X_i$到$X_j$的边，那么在排序中$i$必须小于$j$。由于贝叶斯网络是有向无环图(DAG)，这样的拓扑排序总是存在的。

其实仔细想想，你就按照拓扑排序的结构，从根结点向下走到leaf处就可以了。

但是并不能说爷爷和你就一定没有关系，一定是独立的，或者一定不是独立的，这是一个概率问题，但是如果建立在given parent的情况下，那确实是独立的

这种情况有点相当于是你爷爷和你确实有亲缘关系，但是如果建立在大家都知道你叠是谁的情况下，你爷爷是谁就不重要了，生了你的是你爸妈不是你爷爷

这个很暴论，不过确实可以证明，下一节课会说
## Lec 14
Conditional Independence
- P(x, y) = P(x)P(y), X and Y are Independent
- P(x, y|z) = P(x|z)P(y|z), X and Y are conditionally independent given Z

If the parents of the nodes are not all the previous nodes, we can't represent the nodes in Baye's nodes (Not a Ascylic directed graph).

这边还研究了triple结构的贝叶斯网络，并探讨这些边边是否是独立的

$P(x,y ,z) = P(x)P(y|x)P(z|y)$

Features: 

$P(z|x, y) = \frac{P(x,y,z)}{P(x,y)} = \frac{P(x)P(y|x)P(z|y)}{P(x)P(y|x)} = P(z|y)$

Independent of X given Y.

把Triple各种情况摸清楚，剩下来就是好好理解一下active和inactive各自代表了什么，需要花一些时间阅读记录在notes中的笔记，其实还是有一定思维难度的。

## Lec 15
Inference: calculating some quantities from a joint probability distribution

当然确实也存在不在考虑范围内，但是可能会影响到最终结果的隐藏变量

Inference
- by Enumeration
- by Variable Elimination

本质上是尽早地发现某些变量之间的相关性，降低需要考虑的变量的维度数目

Factor: a multi-dimensional array
- value: P(y1y2..yn|x1x2..xn)
- x1x2..xn is unknown, any assigned X or Y is a dimension missing from the array, while the array, we can enumerate on them

Operation 1: join Factors
- 本质上就是对于存在两个变量的entry，对其进行pointwise products操作
- build a new factor over the union of the variables involved
- 比如，把P(R)和P(T|R)转变为P(R, T)变量，两张表化成一张表

Operation 2: Eliminate
- 直接把某个变量灭掉
- 比如我们不关心R，对于P(R,T)的表，我们直接将其转化成P(T)

第三种方法，我们join一步之后，尽早地去完成eliminate步骤

现在我们大概知道了具体怎么操作，但是还是需要给出一个更加正式的算法
### Algorithm
Input: a bunch of local factors in Bayes' nets

Query: P(Q|E1=e1, ..., Ek=ek)

Start with Initial factors, like local CPTs (but instantiated by evidence)

While there are still hidden variables, not Q or evidence
- pick a hidden variable H
- join all factors mentioning H
- Eliminate H

join all remaining variables and then normalize

这边有一些消除方法并没有用很严谨的方式去做，但是我数学能力有限似乎也不大容易能够证明出来，不过数学确实不是这门课的重点

证明出来了，直接用条件概率的定义就好了，不知道为什么这几个老师根本就没提一句，而是直接观察

the choice for variable ordering counts