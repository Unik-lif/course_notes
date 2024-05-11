# Book review
## Chapter 1
画饼的一章，无所谓
## Chapter 2
搭建实验环境。先前已经在所里服务器中装好了英伟达驱动，输入 nvidia-smi 得到下面的结果：
```
Tue May  7 15:12:34 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.40.07              Driver Version: 550.40.07      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 1080 Ti     Off |   00000000:A1:00.0 Off |                  N/A |
|  0%   41C    P8             19W /  250W |     571MiB /  11264MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A    623811      G   ...sion,SpareRendererForSitePerProcess         70MiB |
|    0   N/A  N/A    729193      G   ...ink/.nutstore/dist/jre/bin/nutstore          2MiB |
|    0   N/A  N/A   1922574      G   /usr/lib/xorg/Xorg                            383MiB |
|    0   N/A  N/A   1922742      G   /usr/bin/gnome-shell                           10MiB |
|    0   N/A  N/A   2413354      G   ...86,262144 --variations-seed-version        100MiB |
+-----------------------------------------------------------------------------------------+
```
跑 CUDA 似乎需要两个不同的编译器
## Chapter 3
host code and device code are different.

Host: CPU 和内存

Device: GPU 和显存

There is nothing special about this; it is shorthand to send host code to one compiler and device code to another compiler. The trick is actually in calling the device code from the host code. One of the benefits of CUDA C is that it provides this language integration so that device function calls look very much like host function calls

对于下面的这一行中<>等标志的解释：
```C
kernel<<<1,1>>>();
```
These are not arguments to the device code but are parameters that will influence how the runtime will launch our device code.

对于 device 中使用的变量，需要特别使用 cudaMalloc 来进行内存分配，之后才能进行使用。

前三章简单地过完了。明天过接下来的三章，争取本周内看完该书。

## Chapter 4
对于其中 julia的解释：

<<<a>>>， 第一个参数指的是 grid,比较好玩，是 blocks 的集合

此外，grid 可以以 dim3 的形式定义，默认是三维的。

BlockIdx.x 与 BlockIdx.y 指向每一个微型计算单元，他们共同组建起来一个 DIM * DIM 这么大的 Grid ，其中的 offset 可以在 BlockIdx 以及 GridIdx.x 的帮助下共同实现，其中 Grid(DIM, DIM)， 因此 GridIdx.x 的值也比较容易获取。

尝试配置了一下 OpenGL ，编译能成功但是 freeglut 的 DISPLAY 有点让我发蒙，算了，我们暂时不跑这些 demo 了，本身我们确实也不太关心图形学的东西。

## Chapter 5
<<<a， b>>>, 第二个参数表示我们希望 CUDA 为每个 block 所创造的 threads 个数。

ab 是用上的线程总个数。

Blocks 实际上是三维的，而 Grid 则是二维世界上多个 Blocks 的集合， CUDA 接受三维输入，会把 Grid 默认视作三维，高为 1。

In the GPU implementation, we consider the number of parallel threads launched to be the number of processors.

After each thread finishes its work at the current index, we need to increment each of them by the total number of threads running in the grid. This is simply the number of threads per block multiplied by the number of blocks in the grid.

本质上是间隔 CPUs 个单元之后寻找下一个用于计算的单元，这边其实就是线程总数，需要比较懂 ILP 就好。

之后有一个叫做 ripple.cu 的例子，看懂了之后对于理解 CUDA 的结构有非常好非常大的帮助，包括 Grid block 之类的东西，使用 offset 之类的东西本质上是为了 linearization,即线性化处理。

### shared memory
看一下示范案例就好，这不难

thread divergence 与 __syncthreads() 不能一起用，前者是让其他线程 idle ，部分线程执行工作。后者则是强制所有线程执行完 __syncthreads,二者的语义是相悖的。

## Chapter 6
这一章讲了 constant 内存的使用方法，相比于一般的内存来说，该 device 内存有良好的同步性：
- A single read from constant memory can be broadcast to other “nearby” threads, effectively saving up to 15 reads.
- Constant memory is cached, so consecutive reads of the same address will not • incur any additional memory traffic.

这边的 nearby 是什么意思？实际上是 warp 的意思：
- a warp refers to the group of threads being woven together into fabric. In the CUDA Architecture, a warp refers to a collection of 32 threads that are “woven together” and get executed in lockstep.

When it comes to handling constant memory, NVIDIA hardware can broadcast a single memory read to each half-warp. 相对于一般的 GPU memory 来说，一个 half-warp 能为 constant memory 节省下 15 次线程读取的开销。

实际上能够节省的开销倍数可能更高，因为 constant memory 的稳定性，最终造成的结果会是， cache 将会激进优化这件事情。

## Chapter 7
texture memory 是另一种特殊的 GPU 内存， texture memory 也是在 chip 之上进行缓存的，因此在某些使用场景下，它能够降低来自于片外 DRAM 的内存访问请求数目，从而提高性能。

GPU 的空间局部性可能范畴拉的要比 CPU 还要大一些，在 CPU 中可能不在同一个缓存行上的东西，放到 GPU 上可能恰好就有不错的空间局部性。

