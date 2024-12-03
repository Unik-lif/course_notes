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
### Uniform Cost Search

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

## Lec4: 
### Graph Search: Never expand a state twice.
Tree Search + set of expanded states ("closed set")

Important: Store the closed set as a set, not a list.

Some features:
- Admissibility: $h(A) \le actual cost h^{*} from A to G$ => $A^{*}$ tree search is optimal
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
