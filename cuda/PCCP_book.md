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