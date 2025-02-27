# Course Instructor: Ion Stoica

## Lec1:
process provides execution environment with restricted rights provided by OS

Nowadays, great gap between computing needs and memory demands and CPU, even GPU can't mitigate this gap.
## Lec2:
Four fundamental OS concepts
- Thread: single unique execution context => fully describes program state
- Address Space: Programs execute in an address space
- Process: Instance of an executing Program
- Dual mode operation: protection, only 'system' can access certain part of resources

Thread: certain registers hold the context of thread. A **thread** is executing on a processor when it is loaded in the processor.

Address Space: Stack, heap.

Multiprogramming: multiple processes of control

Switching processes and threads:
- when switching threads, we only need to change a bunch of registers
- when switching processes, we also need to switch the resources and memory

Concurrency problem => involves resources, OS has to coordinate all activity

The OS will multiplex these abstract machines. While sharing is good, the protection might be compromised.
## lec3:
Talk about process and thread.

Process: a restricted execution environment.

Threads: a single unique execution context

### Motivations for Threads
Operating systems must handle multiple things at once. (MTAO)
### Multiprocessing vs Multiprogramming
multiprocessing: multiple cpus. => parallel

multiprogramming: multiple jobs/processes. => time-slice like， but not parallel

parallelism: doing things simultaneously

concurrency is handling multiple things at once

so concurrency is a boarder concept than parallelism, parallelsim is sufficient to be concurrent.

Threads can compute IO overlap

- How can we make a multithreaded process?
- Once the process starts, it issues system calls to create new threads, these new threads are part of the process, they share its address space.

### Fork-Join Pattern
join will let the father process to wait all the children process.

### Conclusion:
threads are the OS unit of concurrency, an abstraction of a virtual CPU core.

process consists of one or more threads in an address space

we saw the role of the OS library