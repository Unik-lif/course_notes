# 分布式系统
## Lec 1: intro
为什么重要？废话， scaling 是很重要的。

一些关键点
- parallelism
- fault tolerance
- physical
- security/isolated

Challenges:
- concurrency
- partial failure
- performance

Today's paper: Mapreduce.

How to read paper efficiently?

对于论文，我们需要回答问题，并自己提出一些问题

我们将会讨论鸡架：主要讨论三个问题，其一为存储，其二为交互，其三为计算。需要对他们构造抽象。

一些关于部署与实现的事情：
- RPC
- Threads
- concurrency

性能：
- scalbility: 多倍资源可以获得多倍的性能提升

当我们加了足够多的某个设备，性能的瓶颈将会发生转移。

Fault Tolerance
- Availability
- Recoverability

好的容错系统往往在可用性的基础上，还能够允许系统能够被恢复

Usage:
- NV Storage / Replication

Consistency
- 在分布式系统中可能会有多个副本，他们之间要做好同步的工作，尤其是在针对共享数据做修改的时候
- strong/weak guarantee of the consistency
- 不要把鸡蛋放到同一个篮子里/分区容错

Mapreduce
- a system originally built by Google
- scale to millions of servers and computers

