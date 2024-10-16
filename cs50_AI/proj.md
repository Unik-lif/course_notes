## Proj1
### degrees
算最短路径的，感觉是练手项目，把思路整理一下，对部分数据结构做好修改，就能完成。
```python
def shortest_path(source, target) -> list:
    """
    Returns the shortest list of (movie_id, person_id) pairs
    that connect the source to the target.

    If no possible path, returns None.
    """

    # TODO
    # from souce -> target
    frontier = StackFrontier()
    source_node = Node(person=source, parent=None, path=None)
    # target_node = Node(movie=people[str(target)]["movies"], person=target, parent=None)
    print(source_node)
    
    # initialize the frontier and explored set.
    frontier.add(source_node)
    explored = set()

    while True:
        node = frontier.remove()
        # go backwards using node.person to get the (movie_id, person_id) list.
        if node.person == target:
            path = []
            while node.parent != None:
                path.append(node.path)
                node = node.parent
            path.reverse()
            return path

        # add into explored set.
        explored.add(node.person)
        for (movie_id, person_id) in neighbors_for_person(node.person):
            if not frontier.contains_state(person_id) and person_id not in explored:
                neighbor_node = Node(person=person_id, parent=node, path=(movie_id, person_id))
                # print("add node", neighbor_node)
                frontier.add(neighbor_node)

        if frontier.empty() == True:
            return None

    # raise NotImplementedError
```
下面是实验结果。
```
Results for ai50/projects/2024/x/degrees generated by check50 v3.3.11
:) degrees.py exists
:) degrees.py imports
:) degrees.py finds a path of length 1
:) degrees.py identifies when path does not exist
:) degrees.py finds a path of length 2
:) degrees.py finds a path of length 4
:) degrees.py finds a path of length 0
```
### tic-tac-toe
剪枝没有加上，我们先穷搜，以后有机会再做。看起来剪枝要做的，是把递归函数给展开一些，以方便我们提前返回？

或者说，可能要把当前的优选值作为函数的参数输入，以便我们迅速锁定哪些计算是可以跳过的。

完成函数就行了，花不了太长的时间：不过第一次算是搞清楚MinMax的算法的定义，可惜当年写五子棋的时候脑子不大好使。

下面是实验结果：
```
Results for ai50/projects/2024/x/tictactoe generated by check50 v3.3.11
:) degrees.py exists
:) degrees.py imports
:) initial_state returns empty board
:) player returns X for initial position
:) player returns O after one move
:) player returns X after four moves
:) player returns O after five moves
:) actions returns all actions on first move
:) actions returns valid actions for second move
:) actions returns valid actions for fifth move
:) result returns correct result on X's first move
:) result returns correct result on O's first move
:) result returns correct result on third move
:) result raises exception on taken move
:) result raises exception on negative out-of-bounds move
:) result does not change the original board
:) winner finds no winner on initial board
:) winner finds winner when X wins horizontally
:) winner finds winner when O wins vertically
:) winner finds winner when X wins diagonally
:) winner finds no winner in tied game
:) terminal returns True in tied game
:) terminal returns False in initial state
:) terminal returns True if X has won the game
:) terminal returns False in middle of game
:) utility returns 0 in tied game
:) utility returns 1 when X wins
:) minimax blocks immediate three-in-a-row threat
:) minimax finds only winning move
:) minimax finds best move near end of game
:) minimax returns None after X wins
```
## Proj2
### Knight
这个实验很简单，注意model_check函数很有意思，记得仔细理解一下。

把所有的可能性与相关的限制条件列举出来，就能轻松解决这边的需求。
### MineSweeper
> More generally, any time we have two sentences set1 = count1 and set2 = count2 where set1 is a subset of set2, then we can construct the new sentence set2 - set1 = count2 - count1. Consider the example above to ensure you understand why that’s true.

上面的这个部分可能是最关键的信息了。

其实还挺难写的，考虑到`knowledge base`的更新，这确实不是那么容易的，不过找到了一点就可以很好办了。最关键的是不要一口气吃成胖子，这么做就不用太担心状态更新的问题，一步一步来。

计算机的计算能力这么强大，可以一直迭代，直到收敛，然后再继续往下做，这样一来这个就不难做了。

## Proj3
### PageRank
PageRank: the probability that a random surfer is on that page at any gien time.

搞清楚pagerank的原理之后，这个问题就只是一个实现上的问题，并不困难，但可能还是涉及一些细节，需要注意。

其中一个可能会卡住的地方：
- dict不能直接替换，否则将会让其地址设置为一致（没错这个类型其实是地址引用级别的替换，可能做不到深拷贝，而是浅拷贝）
- sample page rank是从其他page衍生过来的，需要注意

### Heredity
逻辑很冗长，但是内容很简单，高中生物课，印象里很多年前写的理综有类似的题目，想清楚了就很好写，唯一的麻烦就是，逻辑有点太长了。

## Proj4 Crosswords
很麻烦，任务量很大，但是能够理解算法的话，不是那么困难，其实就是耗时间！！

```
Results for ai50/projects/2024/x/crossword generated by check50 v3.3.11
:) generate.py exists
:) generate.py imports
:) enforce_node_consistency removes node inconsistent domain values
:) enforce_node_consistency removes multiple node inconsistent domain values
:) revise does nothing when no revisions possible
:) revise removes value from domain when revision is possible
:) revise removes multiple values from domain when revision is possible
:) ac3 updates domains when only one possible solution
:) ac3 updates domains when multiple possible domain values exist
:) ac3 does nothing when given an emtpy starting list of arcs
:) ac3 handles processing arcs when an initial list of arcs is provided
:) ac3 handles multiple rounds of updates
:) ac3 returns False if no solution possible
:) assignment_complete identifies complete assignment
:) assignment_complete identifies incomplete assignment
:) consistent identifies consistent assignment
:) consistent identifies when assignment doesn't meet unary constraints
:) consistent identifies when assignment doesn't meet binary constraints
:) consistent identifies consistent incomplete assignments
:) order_domain_values returns all available domain values
:) order_domain_values returns all available domain values in correct order
:) select_unassigned_variable returns variable with minimum remaining values
:) select_unassigned_variable doesn't choose a variable if already assigned
:) backtrack returns assignment if possible to calculate
:) backtrack returns no assignment if not possible to calculate
```
实在想不出来的话，对着伪代码实现就好了！