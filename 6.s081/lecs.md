## Lec1: Class assign:
- read xv6 book before the class.
- spend some lectures on papers.

## Lec2:
### Strawman Design: no OS, or treat OS as a lib.

This idea is bad cause hirerachy is not set.

A strong reason to have OS: enforce mutli-use and isolation.

Design OS as a Lib is not a common case, maybe in real-time system where all apps are trusted this may be not so bad.

### Unix Interface:
abstracts the Hardware resources.
1. CPU -> process.
2. Memory -> exec.
3. Disk Block -> files.

### OS should be defensive.
Ensure everything work normally.

Apps can't crash the OS. Some OS should be written in a way that its isolation not easily to be broken by apps.

Strong Isolation between apps and the OS. This Isolation always are ensured by the Hardware.

Two basical Hardware supports:
1. user/kernel mode.
2. VM.

#### User/Kernel mode.
unprivileged and privileged instructions.

usually in hardware there is a flag to check the current state.
#### CPUs provide virtual memory.
page table: virtual address -> physical address.

process has own page table. -> completely seperate.
```
---------
|       | -> user mode VM box.
|   LS  |
---------
----------------------------------------
              OS                        Kernel mode
```
```
Entering Kernel: ecall (n) <---- syscall number.
                    |
                    |-----> fork ----> sys_fork -----> trap into kernel (syscall)------> fork
                    |                                         ^       |
                    |                                         |       |
                     -----> wait ----> sys_wait -----> -------        -----------------> wait
```
#### Threat Model
Kernels are sometimes seen as trusted computing base or TCB.

Kernel must treat process as malicious.

about kernel bugs: -> monopolic and micro kernel.

### How does Kernel compiler?
```
       gcc          as
pro.c ----> proc.S ----> proc.o ----> use ld ---> kernel
and etc.
```
Think qemu as a true RISC-V circuit board.