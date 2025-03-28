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

