# OS: typically seen as a resource manager.

## Virtualize
run "./cpu A" below.
Using zsh as possible.
```C
#include <stdio.h>
#include <stdlib.h>
#include "common.h"

int main(int argc, char *argv[])
{
    if (argc != 2) {
	fprintf(stderr, "usage: cpu <string>\n");
	exit(1);
    }
    char *str = argv[1];

    while (1) {
	printf("%s\n", str);
	Spin(1);
    }
    return 0;
}
```
the result is shown below.
```
prompt> ./cpu A & ./cpu B & ./cpu C & ./cpu D &
[1] 7353
[2] 7354
[3] 7355
[4] 7356
A
B
D
C
A
B
D
C
A
...
```
### Virtualizing the CPU:
Even though we just have one processors, with some help from the hard-ware, is in charge of this illusion, that the system has a very large number of virtual CPUs.

### Virtualizing the Memory
Two processes visit same address at the same time independently. As if each running program has its own private memory, instead of sharing the same physical memory with other running programs.

Each process has its own private virtual memory address.

```C
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include "common.h"

int main(int argc, char *argv[]) {
    if (argc != 2) { 
	fprintf(stderr, "usage: mem <value>\n"); 
	exit(1); 
    } 
    int *p; 
    p = malloc(sizeof(int));
    assert(p != NULL);
    printf("(%d) addr pointed to by p: %p\n", (int) getpid(), p);
    *p = atoi(argv[1]); // assign value to addr stored in p
    while (1) {
	Spin(1);
	*p = *p + 1;
	printf("(%d) value of p: %d\n", getpid(), *p);
    }
    return 0;
}
```
the result is shown below.
```
prompt> ./mem &; ./mem &
[1] 24113
[2] 24114
(24113) address pointed to by p: 0x200000
(24114) address pointed to by p: 0x200000
(24113) p: 1
(24114) p: 1
(24114) p: 2
(24113) p: 2
(24113) p: 3
(24114) p: 3
(24113) p: 4
(24114) p: 4
...
```

## Concurrency
Concurrency should be studied with multi-threads.
```C
#include <stdio.h>
#include <stdlib.h>
#include "common.h"
#include "common_threads.h"

volatile int counter = 0; 
int loops;

void *worker(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
	counter++;
    }
    return NULL;
}

int main(int argc, char *argv[]) {
    if (argc != 2) { 
	fprintf(stderr, "usage: threads <loops>\n"); 
	exit(1); 
    } 
    loops = atoi(argv[1]);
    pthread_t p1, p2;
    printf("Initial value : %d\n", counter);
    Pthread_create(&p1, NULL, worker, NULL); 
    Pthread_create(&p2, NULL, worker, NULL);
    Pthread_join(p1, NULL);
    Pthread_join(p2, NULL);
    printf("Final value   : %d\n", counter);
    return 0;
}
```
The final result is 2000 when input 1000, but 109039 when input 100000.

Strange! The adding programs in fact consist of these three steps:
1. load the value of the counter from the memory
2. increase the counter
3. store it back into memory

In multi-threads cases, these three steps don't necessarily execute all at once, the step may be out of order in reality.

## Persistence

Details concerned about store between hardware and software should be taken into considerations.

## Design Goals:

1. abstractions: easy to use
2. performance: minimize the overheads
3. protection: Isolation or sandbox
4. high degree of reliability: run a long time.
5. energy-efficiency, security, mobility, etc...

main idea: takes physical resources, such as a CPU, memory or disk and virtualizes them. Handles tough and tricky issues related to concurrency, stores file persistently.
## Histroy:

### Libraries: batch processing.-----> file system(system call)

Diffrence betwwen a system call and a procedure call: 
    
    system call: transfer control into the OS while simultaneously raising the hardware privilegd level.

    user mode: restricts.  <-----------------------------------return from trap instruction
                                                                       |                                            
    kernel mode: full access.(use trap handler to raises the privilegd)------> done


### Multiprogramming

desire to make better use of machine resources.

e.g.: Unix or Unix-like systems.

### Modern Era

PC

## Processes
process ensence: running program->process.

to implement virtualization of the CPU, the os will need both low-level machinery and some high-level intelligence.

low-level mechanism: e.g. context switch.
### abstraction:
what constitutes a process, we should understand the machine state.
1. memory.
2. registers.
3. I/O devices.

policy -> high-level, mechanism -> low-level.

we'd better separate them, use modularity more. this is a general software design principle.

for process, 5 instructions of OS should be considered.
1. create
2. destroy
3. wait
4. miscellaneous control
5. status

### process creation:
load programs initially reside on disk onto memory, allocating address for them.

other work should be done by OS:
1. run-time stack (set up for local variable, function parameters, and return addresses).
2. heap: for dynamically-allocated data (function malloc() free())
3. I/O work: Unix file descriptors
PS: this process is done lazily now, not load until when be used at once. (due to paging and swapping).
### process states:
```
         descheduled
  running -------------> ready
     \    <------------- /
      \      scheduled  /
       \               /
I/O init\             / I/O done
            blocked 
```
in fact, besides these three states, in some system like Unix, we also get initial state and final state.

the final state is aka zombie state.

for processlist, it contains info about all processes in the system. Each entry is found in what is sometimes called a process control block (PCB). It is just a structure that contains info about a specific process.

### HW:
#### q1:
100 % CPU utilization. because I/O is not involved.
#### q2:
5 + I/O time.
#### q3:
1 + Max(4, I/O time).

in fact, it counts.
#### q4:
5 + I/O time
#### q5:
same as q3.
#### q6:
run all cpu processes out, then I/O process.
#### q7:
act like insert.

cause every time io will take a lot of time.
#### q8:
random result.

### API code work
Done.

## Charpter 6
virtualize the CPU is based on a very simple idea: run one process for a while, then run another one, and so forth. This is AKA time sharing.

Two problems should be solved before we adapt virtualization:
1. performance
2. control
### basic Technique: limited direct execution
direct execution: just run the program directly on the CPU.

what OS and Program do is shown below:
```
OS                                             Program
-----------------------------------------------------------------
create entry for process list
allocate memory for program
load program into memory
set up stack with argc/argv
clear registers
execute call main()
                                        run main()
                                        execute return from main
free memory of process
remove from process list
```
if we just run a program, how can the OS make sure the program doesn't do anything that we don't want it to do.