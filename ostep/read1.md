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

so on.