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