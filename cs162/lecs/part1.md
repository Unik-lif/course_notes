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

### Protection
Base and Bound

 
### Motivations for Threads
Operating systems must handle multiple things at once. (MTAO)
### Multiprocessing vs Multiprogramming
multiprocessing: multiple cpus. => parallel

multiprogramming: multiple jobs/processes. => time-slice like， but not parallel

parallelism: doing things simultaneously

concurrency is handling multiple things at once

so concurrency is a boarder concept than parallelism, parallelsim is sufficient to be concurrent.

concurrency可以是线程多个进行切换，而parallelism必须同时处理

Threads can compute IO overlap
- 等到I/O做完了之后再重新切换回来

- How can we make a multithreaded process?
- Once the process starts, it issues system calls to create new threads, these new threads are part of the process, they share its address space.

### Fork-Join Pattern
join will let the father process to wait all the children process.

### Conclusion:
threads are the OS unit of concurrency, an abstraction of a virtual CPU core.

process consists of one or more threads in an address space

we saw the role of the OS library
## Lec4:
Interleaving and Nondeterminism

Due to the same address space, the value of the parameter might be changed by other threads.
### Relevant Definitions
Multual Exclusion: Ensuring only one thread does a particular thing at a time.

Critical Section: code exactly one thread can execute at once.

Lock: An object only one thread can hold at a time.

可以通过semaphore信号量的消耗与增长来落实ThreadJoin
### File abstraction
Unix/POSIX Idea: Everything is a 'File'

## Lec5:
Files Abstraction
- High Level File I/O: Streams
- Low Level File I/O: File Descriptor

Kernel Buffering
- Reads are buffered inside kernel
- Writes are buffered inside kernel

Reasons:
- Always read blocks from the device
- Wait for rotation of the device

Duplicating Descriptors

### High-Level and Low-Level File API
fread vs read: The fread will do more work in the user world.

Operations on file descriptors are visible immediately (The buffer is in the kernel), but streams are buffered in user memory. The user memory is in application buffer. Streams are buffered in user memory.

'\n' in the printf => indicate the end of the line => write user buffer to the kernel buffer.

### FILE Buffering
FILE Buffer, if the buffer is full, then it is flushed, which means it's written to the underlying file descriptor

```C
char x = 'c';
FILE * f1 = fopen('file.txt', 'w');
fwrite('b', sizeof(char), 1, f1);
// > We'd better put a fflush(f1) here to make it deterministic.
FILE * f2 = fopen('file.txt', 'r');
fread(&x, sizeof(char), 1, f2);
```
each of the f1, f2 will create a file buffer, but we can't ensure that the write to the buffer (i.e. 'b'), has made it to the driver disk.

> Why buffer in userspace? Overhead!!

syscalls are 25x more expensive than function calls.

> Why buffer in userspace? Functionality!

system call operations less capable => simplifies operating system


For each process, kernel maintains mapping from file descriptor to open file description.

### Pitfalls
> Don't fork in a process that already has many processes.

only the thread that called fork() exists in the new process

> Mindful when you use low level and high level file I/O, they are quite different.

high level IO will have a user buffer.

### IPC and Sockets
Key Idea: communication between processes and across the world looks like File I/O

