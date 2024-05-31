## 实验概览
在多核情况下重新设计数据结构以避免忙等。

> The basic idea is to maintain a free list per CPU, each list with its own lock. 
- 需要为每一个 CPU 都维持一个 free list , 每个 list 都有他们用于操作的自己的 lock 结构
> Allocations and frees on different CPUs can run in parallel, because each CPU will operate on a different list. 
- 如果把上面这件事做了这件事情应该不麻烦
> The main challenge will be to deal with the case in which one CPU's free list is empty, but another CPU's list has free memory; in that case, the one CPU must "steal" part of the other CPU's free list. Stealing may introduce lock contention, but that will hopefully be infrequent.
- 特殊情况，一个 CPU 的 free list 是空的，即当前该 CPU 视角下内存已经被用尽。但是另外一个 CPU 则是有空闲内存。需要让前面的 CPU 能够盗取另外一个 CPU 的空闲队列用来使用。


### 实验记录：
- Memory allocator

> 卡在了一个莫名其妙的地方：我单知道要避免找自己了，但是居然忘记 for 的语义了，是不是代码写的太少了？

对于这个，我的理解应该是没有什么问题的，主要还是因为卡在莫名其妙的地方。严格记录一下，我在低级错误大全之中加入了一个模块，引以为鉴。