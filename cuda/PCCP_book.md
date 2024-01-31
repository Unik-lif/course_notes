## Chapter 1: Heterogeneous parallel computing with CUDA
Two distinct areas of computing technologies:
- Computer Architecture(hardware aspect)
- Parallel Programming(software aspect)

Havard architecture: 存算分离
```
                               CPU
                              |------------------------|
                              |                        |
----------------------        |  Arithmetic Logic Unit |
| Instruction Memory | <----> |           ^            |
----------------------        |           |            |
                              |      Control Unit      | <------> Data Memory
                              |------------------------|
                                          |
                                          |
                                      Input/Output
```
### Some Terminology:
Parallel Program: any program containing tasks that are performed concurrently is a parallel program.

Task: when a computational problem is broken down into many small pieces of computation, each piece is called a task.

Parallelism:
- Task parallelism: when there are many tasks or functions that can be operated independently and largely in parallel. Distribute functions across multiple cores.
- Data parallelism: when there are many data items that can be operated on at the same time. Distribute data across multiple cores. => what CUDA tends to handle.

in Data parallelism:
- block paritioning: many consecutive elements of data area chunked together, each chunk is assigned to a single thread in any order, and threads generally process only one chunk at a time.
- cyclic partitioning: fewer data elements are chunked together. Neighboring threads receive neighboring chunks, and each thread can handle more than one chunk.
### Arch:
- SISD: traditional computer, a serial arch.
- SIMD: multicores, execute the same instruction stream at any time, each operating on different data streams.
- MISD: each core operates on the same data stream via separate instruction streams.
- MIMD: multiple cores operate on multiple data streams, each executing independent instructions.

Terminologies in Arch to reflect the abilities of a system.
- Latency: time it takes for an operation to start and complete.
- Bandwidth: **amount of data** that can be processed per unit of time. => MB/sec or GB/sec
- Throughput: **amount of operations** that can be processed per unit of time => gflops. => billion floating-point operations per second.

Two main types:
- Multi-node with distributed memory
- Multiprocessor with shared memory

GPU-like models: Summarized as SIMT(single instruction multiple thread)
- multithreading
- MIMD
- SIMD
- ILP

CPU core vs GPU core
- CPU: optimize the execution of sequential programs
- focusing on the throughput of parallel programs
### Heterogeneous computing:
instead uses a suite of processor arch to execute an application, applying tasks to arch to which they are well-suited.

A heterogeneous application consists of two parts:
1. CPU: host => host code
2. GPU: device => device code
3. They use PCIe Bus to communicate.

To better our performance, we'd better run sequential portion in CPU, and run Compute intensive portion in GPU. A programming model named CUDA is designed to support joint CPU + GPU execution of an application.

The thread in GPU and CPU is a little different, the GPU thread is a relative a lighter one.

### CUDA hierarchy
- Memory hierarchy structure

- Thread hierarchy structure

Three key abstractions:
1. a hierarchy of thread groups
2. a hierarchy of memory groups
3. barrier synchronization

## Chapter 2: CUDA Programming Model
For a Programmer, view parallel computation from different levels:
1. Domain Level: decompose data and functions so as to solve the problem correctly and efficiently while running in a parallel environment
2. Logic Level: 
3. Hardware Level
### Programming Structure
NVIDIA introduced a programming model improvement called Unified Memory starting with CUDA 6, which allows you to access both the CPU and GPU memory using a single pointer.

A typical processing flow of a CUDA program follows this pattern:
1. Copy data from CPU memory to GPU memory
2. Invoke kernels to operate on the data stored in GPU memory
3. Copy data back from GPU memory to CPU memory

### Managing Memory
GPU has two major ingredients in memory structure: global memory and shared memory.

global memory => CPU system memory => shared in One Grid

shared memory => CPU cache => shared in One Block

Grid => Blocks => Threads
1. blockIdx: block index within a grid
2. threadIdx: thread index within a block

The coordinate variable is of type uint3, a structure containing three unsigned integers with x,y,z fields respectively.

The dimensions of a grid and a block are specified by following two built-in variables:
1. blockDim: block dimension, measured in threads.
2. gridDim: grid dimension, measured in blocks.

Two distinct sets of grid and block variables in a CUDA program:
1. manually-defined dim3 data type: only visible on the host side. => Judge the dimension of Grid and Block.
2. pre-defined uint3 data type: only visible on the device side.
### Launching a CUDA kernel
A CUDA kernel call is a direct extension to the C function syntax that adds a kernel's exeuction configuration inside triple-angle-brackets.
```C
kernel_name <<<grid, block>>>(argument list)
```

By specifying the grid and block dimensions, we configure:
1. the total number of threads for a kernel
2. the layout of the threads you want to employ for a kernel