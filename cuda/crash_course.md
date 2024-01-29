## Lec1:
the SIMT model: single instruction multiple threads.
### Some Terminology:
Threads: lowest granularity of execution, executes instructions.

Warps: lowest schedulable entity in SIMT, got a cluster of threads, executes instructions in lock-step, not every thread needs to execute all instructions.

Thread Blocks: lowest programmable entity, is assigned to a single shader core, can be 3-D.

Grids: how a problem is mapped to the GPU, part of the GPU launch parameters, can also be 3-D.

