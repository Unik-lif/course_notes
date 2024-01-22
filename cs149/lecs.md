出于本次项目需求，需要快速搞清楚`CUDA`的一些东西。

快速过一下五节课应该就可以有一个大概的了解，需要对部分名词有一个基本的认知，不然感觉工作有点难进行。

## Lec1:
三个`demo`体现了不同的并行特性：
1. 并行确实对工作有加速
2. 多人并行时存在`running idle`，需要合理分配任务
3. 量级上来之后，`communication`的开销会比计算任务本身还要更大。

课程主题：
1. big scale设计
2. how parallel computers work
3. Fast != Efficient, Thinking about efficient. Is 2x speedup on computer with 10 processors a good result?

ILP: instruction level parallelism

Power Wall. => heat and energy bottleneck => multi-core and parallelism
## Lec2:
Idea1: Use increasing transistor count to add more cores to the processor.

Idea2: Amortize cost/complexity of managing an insruction stream across many ALUs

### Some terminology:
1. Instruction stream coherence: same instruction sequence applies to all elements operated upon simultaneously, necessary for efficient use of SIMD processing resources.

- e.g: a pthread act with no if-else condition, therefore SIMD instructions can work well, can do multiple things at the same time without harmming the correctness.

2. Divergent execution: 与上面的情况相反，有很多if-else判断，由于每个线程走的分支都有较大不同，使得SIMD指令的表现效果非常不好

3. Memory Latency: the amount of time for a memory request from a processor to be serviced by the memory system. e.g. 100 cycles, 100 secs.

4. Memory bandwidth: the rate at which the memory system can provide data to a processor. e.g. 20 GB/s

5. stalls: wait for I/O, memory accessing time latency.
  
Multi-core: use multiple processing cores.

SIMD: use multiple ALUs controlled by same instruction stream (within a core)

Superscalar: exploit ILP within an instruction stream.

prefetching: hide latency. predict.

cache: cache can also have a bandwidth improvment. (bigger bandwidth in L1 cache and CPU)

### Hiding stalls with multi-threading
when stall => switch to another thread.

Key idea: share the core in multi-threading. 对于一个整体，throughoutput变大了，因为处理器一直都在工作，但是对于单个任务来说，需要执行的时间变长了。因为需要重新轮到他，在stall结束后，不一定马上就轮到他了。

### GPU与CPU
GPU: small caches, many threads, huge memory BW, rely mainly on multi-threading

CPU: big caches, few threads, modest memory BW, rely mainly on caches and prefetching

GPU长于计算，但是访存能力比较弱，这边是CPU的Cache做的更精细。
## Lec3:
### Review: 这边的总结很好
a very simple processor:
```
-----------------
|  Fetch/Decode |
-----------------

----------------
|     ALU      |
----------------

---------------------
| Execution Context |
---------------------
```
为了让这个过程更快，可以添加一些新的部件，让处理器可以从一个指令流中，在每个时钟周期中运行两条指令。这个叫做超标量处理器设计。

```
-----------------          -----------------               
|  Fetch/Decode |          |  Fetch/Decode |
-----------------          -----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

            ---------------------
            | Execution Context |
            ---------------------
```

更好的方案 => dual-core processors.

再更进一步：multi-core + superscaler

其他更新：SIMD => one instruction with multi ALU.
```
            -----------------
            |  Fetch/Decode |
            -----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------


            ---------------------
            | Execution Context |
            ---------------------
```
SIMD with multi-threaded cores (more execution context, add more concurrency to the structure)

在这边，有更多的线程之后，可以避免`memory stall`的情况。
```
            -----------------
            |  Fetch/Decode |
            -----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------


---------------------       ---------------------
| Execution Context |       | Execution Context |
---------------------       ---------------------
```
`Hyperthreading`是`superscalar`和`multi-threaded`的结合。

下面这个模型是四个有超标量性质，支持SIMD指令，并且还是多线程的处理器核。
```
-----------------               -----------------
|  Fetch/Decode |               |  Fetch/Decode |
-----------------               -----------------



----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |          ------------------------------------------
----------------           ----------------          |                                        |
                                                     |                Exec   1                |
----------------           ----------------          |                                        |
|     ALU      |           |     ALU      |          ------------------------------------------
----------------           ----------------

----------------           ---------------- 
|     ALU      |           |     ALU      |
----------------           ----------------



---------------------       ---------------------
| Execution Context |       | Execution Context |
---------------------       ---------------------
```
One decode has to be regular instructions, and one should be SIMD instructions.

### abstraction vs implementation
ISPC way. I simply kind of skip this part.

shared address space model of communication

Message passing model

## Lec4:
