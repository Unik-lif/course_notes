# abstraction
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