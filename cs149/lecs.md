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

#### SPMD: programming abstraction
call to ISPC function spawns "gang" of ISPC program instances.

All instances run ISPC concurrently.

upon return, all instances have completed.

When we are doing coursework, we should review this part.

shared address space model of communication 

Message passing model

Some ISPC keywords:
1. programCount: number of simultaneously executing instances in the gang.
2. programIndex: id of the current instance in the gang.
3. uniform: a type modifier, all instances have the same value for this variable. Its use is purely an optimization, not needed for correctness. statically this type will have the same value.
4. foreach: declares parallel loop iterations: programmer says: these are the iterations the instances in a gang cooperatively must perform. => the underlying ISPC will interleaving, therefore can do quickly.

interleaving is a better way to use SSE instruction

abstraction => programmer's view

implementation => SIMD micro structure.

Interesting ISPC details:
1. uniform => one copy, can't add non-uniform variables.
2. ISPC compiler directly access the resource of the CPU.
## Lec4:
data-parallel model: same operation on each element of an array.

add(a, b, n): instruction on vectors a, b of length n.

map-reduce: SPMD programming.
- map(function, collection)
- function is applied to each element of coleection independently
- SPMD: synchronization is implicit at the end of the map

cannot write a non-deterministic program.

### Gather/Scatter: two key data parallel communication primitives.
These instructions are supported with AVX2 in 2013.

Just do like their meaning in verb forms.

### Process of a parallel program.
problem to solve => decomposition into subproblems => Assignment these kind of work to parallel threads(aka. workers) => Orchestration with parallel program (communicating threads) => mapping: exeuction on parallel machine (come to hardware).

#### Decomposition:
what kind of work?

main idea: create at least enough tasks to keep all execution units on a machine busy.

to avoid ILP obstacles, the key aspect of decomposition is, identifying dependencies.

##### Amdahl's Law: 
in fact this law also describes the dependencies limit maximum speedup due to parallelism.

because some part of the programs are born-to-be sequential, if we denote `S` as the fraction of sequential execution that is inherently sequential, then maximum speedup due to parallel execution should never exceeds `1/S`

adding P result (can be parallel easily, but no need), in common case without parallelism, ILP is always one during the process (one add per clock).

#### Assignment:
`foreach` in ISPC is a very good example.

also a dynamic assigment method can be enabled by ISPC. => worker pools might be a quicker way when we got many small works to do.

#### mapping to hardware
1. pthread: OS.
2. ISPC: compiler
3. CUDA: gpu mapping.

#### Example: grid solver algorithm.
Convert an sequential algorithm into a parallel one using gauss seidel methods.

red-block coloring. (think about cache locality and the communication it involved.)

#### Barrier synchronization primitive.
lock is a way do as a barrier to synchronize.

## Lec5:
### GPU and CUDA.
What is the difference in GPU programming?

- key entities of GPU: describe things from veritices/primitives/fragments/pixels.
- using list_of_positions to rendering a picture.
- - input a list vertices in 3D space
- - group vertices into primitives
- - generate one fragment for each pixel a primitive overlaps
- - compute color of primitives for each fragment
- - put color of the closest fragment to the camera in the output image
### Tips on how to explain a system
- step1: describe the things (key entities) that are manipulated (nouns)
- step2: describe operations the system performs on the entities (verbs)

### Programming:
OpenGL API: graphics programming APIs provided programmer

but now the world found out vertex processing and fragment processing should be programmable.

GPUGPU: general purpose GPU.
### an Alternative way as GPU
you can simply take it as a special CPU now.
### CUDA programming language
"C-like" language to express programs that run on GPUs using the compute-mode hardware interfaces.

CUDA hierarchy: one grid, x blocks; one blocks, y threads.

CUDA kernel: for each thread computes its overall grid thread id from its position in its block (threadIdx) and its block's position in the grid (blockIdx)


Interestingly, the "threads" in CUDA are numbered in a two-dimensional space. This means that instead of being given a "threadID," they are given an x and y location corresponding to where they are with regards to the blocks and such.
### an example to understand:
```C
const int Nx = 12;
const int Ny = 6;

dim3 threadsPerBlock(4, 3, 1);
dim3 numBlocks(Nx/threadsPerBlock.x, Ny/threadsPerBlock.y, 1);

matrixAdd<<<numBlocks, threadsPerBlock>>>(A, B, C);

__global__ void matrixAdd(float A[Ny][Nx], float B[Ny][Nx], float C[Ny][Nx])
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    int j = blockIdx.y * blockDim.y + threadIdx.y;

    C[j][i] = A[j][i] + B[j][i];
}
```
threads id in CUDA can be up to 3-dimensional.
### CUDA execution model:
#### memcpy primitives exist:
```
Host
-------------------------------            ----------------------------------------
|                             |            |                                      |
|  Host memory address space  |----------- | Device "global" memory address space |
|                             |            |                                      |
-------------------------------            ----------------------------------------
```
GPU can use cudaMalloc, cudaMemcpy to operate memory accessing process between the CPU and the GPU. Of course we can't derefence the memory in GPU.
#### Memory model
```
readable/writable by all threads in a block: per-block shared memory => one block of a Grid.

readable/writable by thread: per-thread private memory => one thread of a Block.
```
Every thread in common case only operate in one element. One threads correspond to a[i][j]

Think about the CUDA like below:
the CUDA is consist of a bunch of blocks, and a block consist of threads, and the threads work together to get a newer version of block.  

warp: an instruction stream, aka thread groups (in NVIDIA GPUs groups of 32 CUDA threads share an instruction stream, these groups called warps) , every clock the GPU have the ability to use warp selector to choose several warps to run.

using warp therefore 32-wide SIMD can put into use.

one block is consist of a group of warps!!

try to analysis the whole gpu!

这一课的最后一小部分还不是特别懂，之后需要重新再看一遍，以加强理解。

太累了，总算补足一些`CUDA`就并行上的知识漏洞了，两天刷了`5`节一个半小时的课，心力交瘁。