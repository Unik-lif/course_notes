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

## Lec3: Page Table.
One of the main features that Page Table that provide is the Isolation.

How to do this? -> Address Space.

Give every apps its own address spaces, including the Kernel.

CPU transfers VA to MMU, which will map VA to PA. Every Map (The Page Table) will be stored in Memory, MMU simply go through the Memory.

SiFive manual => riscv qemu hardware version.
## Lec4: RISC-V Convention
RISC-V ISA is a modularize one, which means we can divide it into different modules, and select part of them on demand.

`apropos` is an interesting thing that can help us to refer gdb instructions quickly.

RISC-V also has compressed version, where `s*` registers are reduced.

caller register => not preserved across function call.

callee register => preserved across function call.

so callee register during the calling process should restore the prior values, while caller register don't have to.

我的意思是，让函数能够正常返回的`stack`机制是由操作系统来决定的还是由硬件来决定的？我在`xv6`中并没有看到操作系统决定之事的源代码，这看起来调用一个函数，并且在结束之后正常返回的机制是硬件实现的？

这个过程似乎是由硬件来实现的，啊，真的好好奇，希望自己动手做一遍。

using gdb command like `bt` and `frame n` and i `frame`, we can get the info of the stack frame.
## Lec5: Traps
What you can do in Supervisor mode but not can do in User mode is not so much.

for example, supervisor mode is still restricted to the page table where `satp` register sets.

some gdb skills:
1. `ctrl a + c` to enter qemu console.
2. `info mem` to see memory page table in qemu.