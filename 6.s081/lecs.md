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
## Lec5:
### Preclass
Calling Convention:
- riscv a0-a7 eight integer registers for calling convention passes arguments in registers when possible. fa0-fa7
- 不同的数据结构在传输时会根据pointer-size来进行对齐，最多可以直接传两个pointer-size长度的参数，更长的则需要通过地址引用来做。
- 函数返回值则通过`a0`和`a1`两个寄存器，更大的返回值会整个在内存中进行传输。
```
Larger return values are passed entirely
in memory; the caller allocates this memory region and passes a pointer to it as an implicit first
parameter to the callee
```
- riscv中的栈是十六字节对齐的，需要注意
- riscv中的temporary寄存器为s0-s11，他们需要在函数调用的前后维持同样的值，所以在进入函数之后适时需要做压栈。
- 但是t0-t6可以随便用，这个东西由caller来进行管理
### Class
What you can do in Supervisor mode but not can do in User mode is not so much.

for example, supervisor mode is still restricted to the page table where `satp` register sets.

some gdb-qemu skills:
1. `ctrl a + c` to enter qemu console.
2. `info mem` to see memory page table in qemu.
3. `i frame`
4. `p *argv@2`：展示数组的两个元素
5. `i locals`
caller registers => Not perserved across fn call (can be overwritten)

callee registers => preserved across fn call (can't be overwritten, should be saved)

a0, a1 <= return a long long value.

The stack frame of RISCV is nearly the same as x86-64, at least in my point of view I don't consider them different.

## Lec6: Traps

```
SH
       write()
       ecall  <------------------------>
---------------------------            |
       uservec       trampoline     userset()
              |                        |
           usertrap               usertrapret() 
              |                        |
            syscall ------------------>
              ^  >
              |  |
            sys_write
```
xv6在系统调用的时候，会在用户地址空间和内核地质孔建的trampoline中的trampframe之中对相关寄存器进行存储。

嗯，理论上我们NestedSGX的trampoline也应该留有一个页干这个事情，不过我似乎已经规避了这个风险。。我们似乎没有按照正统的方式来做，而是用了相对hacky的方式来实现这个事情。

### 课后作业：
自己过一遍课上讲的系统调用干的全流程，请用gdb调试，不要自己硬看。

1. 为什么这些应用程序可以找到系统调用位置？`usys.S`是怎么连接上的？
- 对照Makefile阅读，当然你可以加上-nB选项，会非常清晰。
- riscv64-unknown-elf-ld -z max-page-size=4096 -N -e main -Ttext 0 -o user/_stats user/stats.o user/ulib.o user/usys.o user/printf.o user/umalloc.o user/statistics.o
- 可以看到在链接的是后把usys.o都静态链接进去了，因此系统调用是可以找到调用入口的

2. 为什么在`stepi`中调用`ecall`时没法跟踪到内核？
- 确实这个现象比较奇怪，不过`stackoverflow`上有人也提了一下这个问题。
- https://stackoverflow.com/questions/60795578/cannot-access-kernel-space-when-debugging-xv6-with-qemu-and-gdb，我们最好自己编译一遍工具链，感觉可能是早期riscv与ubuntu20.04绑定的工具链包还不够稳定的问题。
- 不过这个似乎还挺考验网速的，正好我要去一趟玉泉路。
- 在我们的服务器上配置好了这个xv6的环境，不过服务器自己用的qemu肯定不是现在这个，需要仔细一点配，并且修改一下使用的路径，现在用起来会很舒服。

看完了第四章。
## Lec7: Page Faults

Plan: implement VM => features using page faults.

more in design level, less in code level.

Virtual memory benefits
1. Isolation
2. Level of indirection: va -> pa.
- using page faults, we can dynamically change the mappings in VM. => great flexibility.

Info needed:
1. the faulting va. <= stval register
2. the type of page fault. a number of causes involved. <= scause register
3. the va of instruction that caused the fault. <= sepc register. trapframe->epc

allocation type:
1. sbrk => so long as the application asks how many space, we simply allocates the space. (eager allocation)
- 这个方式是 xv6 源代码本来的写法
2. pagefault => demanding page. p->sz, p->sz+n. 直到出现了 PF ，我们才尝试去做页表映射这件事
- 这个方式是我们未来希望做修改最终希望看到的写法

#PF 的标准处理流程：
1. 再分配一个页
2. 对于这个页做清空
3. 映射这个页
4. 重新开始运行先前的指令

在演示视屏中出现的 uvmmap: panic 似乎源自 unmap 从较低的地址到较高地址的连续释放，虽然 pagefault 能够 mmap 掉一部分，但是不代表所有的内存都是这样，有部分内存那确实是 lazy-allocation ，并没有真正地被分配内存。

### 延伸：copy-on-write 和 demanding pageing
一样是在 pagefault 发生的时候，即权限不对劲。
- COW： 于是把原本的东西拷贝一部分，然后再附上我们新的权限，再对东西做修改。
- DP： 从磁盘中读取出来

memory-mapped files
- mmap -> 直接在内存上改
- unmap -> 需要把东西从内存读回到文件之中
- file will be one of the vma.

page tables      ->  
                            great VM features.
traps/page fault ->

## Lec9: Interrupts.
Basic idea: 
- the HW wants attention now.
- the SW save its work => process interrupts => resume its work.

Same mechanism with syscall + traps.

only slightly different, the differences are:
1. asynchronous.
2. concurrency
3. programmable devices

Where do interrupts come from?
- PLIC: platform level interrupt controllers.

Driver manages device.

Programming device. => memory mapped i/o => use load/store instruction onto control register of the device.

Tips:
- read the manual carefully.
- follow the protocol.

### Case Study.
How can we print into the console?
- device puts char into uart, uart generates interrupts when the char has been sent.
- keyboard receive the line and generates the interrupts to tell the processor that here's a char capable on the keyboard.

RISCV supports for interrupts:
- SIE: one bit for External/Software/Timer interrupts.
- SSTATUS: bit to enable/disable interrupt
- SIP: interrupt pending
- SCAUSE:
- STVEC

我发现似乎有点跟不上老师的节奏了，或许我应该自己先调研一下相关的代码。

一些方法论上的东西：
- 首先对于驱动，要对其做初始化，比如 UART 串口需要开启其对于字符串读取的功能
- 其次，使用 PLIC 来管理各个 CPU 核对于中断的敏感度，使得各个 CPU 有能力对部分特殊异常产生兴趣，到这里就实现了驱动以及外设对于中断的外部准备
- 最后，设置 CPU 的 SSTATUS 寄存器，使得 CPU 真正地能运转起中断来

Producer / consumer

Interrupt evolution:
- interrupt used to be fast, hardware that time is simple.
- now interrupt is slow, many steps should be done if a interrupt is happend. Since device is more complicated now.
- 考虑一种高速网卡的情况，如果持续处理， CPU 甚至会跟不上中断处理的速度。

Solution to fast devices interrupts: polling. 轮询
- cpu keeps reading RHR register. CPU spins. Not dependent on the interrupts.

对于部分高级一些的网卡驱动，可能还会动态在轮询和终端之中进行切换。

## Lec 10: Multiprocessors and locks.
app wants to use multiple cores. Kernel must have abilities to handle parallel syscalls.

We need locks for correct sharing. Locks can limit performance.

Why locks? Avoid races conditions.

Lock Abstraction:
```C
qcquire(&l)
// critical section
release(&l)

 struct lock {

}
```
如果程序有多个锁，那么我们的平行化也会因此做的更加好。

当然有部分大程序可能是 lock-free 的，它们会更加复杂，但是性能自然也会更加好。

对每个对象自动加锁（单位锁）不见得是一个好事情，可能还会有一些原子性没做好的地方。然而，如果对于我们所需要的全部资源都加锁，反而会有更大开销以及比较差的并行性能。

lock perspectives:
- locks help avoid lost updates.
- locks make multi-step operations atomic.
- locks helps maintain invariant.

Deadlock: deadlock embrace. 

one solution: order that locks. full operations have to acquire locks in that order.

虽然我们没有必要对于所有的锁都做一定的排序，但是当我们比较两个锁的优先级之时，我们要有能力能够比较出来。
### Lock vs modularity
lock ordering must be global => should be aware of other locks.

but it kinda break the abstraction that m2 shouldn't leak lock info to m1.

### Lock vs performance
need to split up data structures to get bet better performance.

Best split is a challenge. may have to rewrite the code if you want to split up.

general receipes:
- start with coarse-grained locks
- measure whether a contention that appears one of these locks.

### Hardware assisted.
in Riscv -> amoswap: atomic memory operation swap.


amoswap addr, r1, r2
```
lock addr
tmp <- *addr
*addr <- r1
r2 <- tmp
unlock
```
r2 can be the return value in fact... if we set r2 to a0.

afterwards, *addr will be written to r2, r1 will be written to addr, this is a very good example to show `test_and_set` case.

It is actually dependent on how the memory system actually works.

lock的作用：
- 单 CPU 因中断等其余行为出现的同步问题
- 多 CPU （多线程） 同步问题

### Memory ordering
假设我们先设置 locked 为 1, 再设置 x = x + 1 ,再解锁。在程序顺序时，也就是说如果我们只有一个串行执行的环境下，即便代码顺序乱了，我们一样可以允许 CPU 和编译器对代码进行优化，以获得更好的性能。

但是在多个核的情况下，可能给优化乱掉，如何避免这个优化问题？一些基本的原语：
- sync_synchronize: act as memory fence.

本质上达到的效果如下：

```python
inst1..n
fence # fence1
inst2n..3n
fence # fence2
instn..2n
```
inst1..n 不会因为优化越过 fence1 在 fence1 之后执行，inst2n..3n 不会越过 fence1 和 fence2 在区域以外执行， instn..2n 不会在 fence2 以前执行。

总结：
- 锁对于多核来说是个保证程序正确性的好东西，但是会降低性能
- 锁会把程序复杂化
- 在不需要共享的情况下不要做共享以及加锁处理
- 需要用锁的时候，先从粗粒度做起，再尝试把粒度精细化
- 多使用 race detecter

特别的， sync 指令不仅对硬件有效，对于编译器一样有效。所以 sync 是个好东西。

riscv 特权级手册对于内存模型有一个很详细的讨论，也会讨论编译器的行为，值得去看一下这一章。
## Lec 11:
Thread - one serial execution. only use one CPU.

Thread has pc, regs, stack.

Interleave threads:
- have multiple cpus
- switch system.

Shared memory?
- xv6 kernel threads do share memory.
- xv6 user processes: no shared memory among these threads, they have their individual space.
- linux users -> allow shared memory

Thread Challenges
- switching - interleave => scheduling, how to pick next thread.
- what to save / restore.
- compute - bound how to handle.

Timer interrupts -> kernel handler -> yields - switch.

Preemptive sched

"state" of threads:
- running
- runnable <=== PC, registers.
- sleeping

xv6的线程切换：
- 一个用户进程先跳入到内核之中，保存用户线程的状态 context ，并且运行用户进程的内核线程，把自己挂起
- 切换到另外一个进程的内核线程，然后这个新的进程再切换到自己的用户线程来跑
- 最终的效果是从一个用户进程切换到另外一个用户进程

更加精细一些：实际上 ctx 的切换是切换到 scheduler 的 ctx ，之后再由 scheduler 切换 ctx 到另外一个用户进程的 ctx 之中。每个 scheduler 实际上有一个单独的栈。

对于 xv6 来说，每个进程一共有两个线程，一个是用户线程，一个是内核线程，当然他们王不见王。

Switch为什么没有保存全部的状态？
- 它本质上是一个 C 函数，我们只需要保存好 callee 寄存器的值就好了

## Lec 13: 
在 xv6 中，任何东西调用 swtch 时，我们总是让其预先获取这个线程的锁，然后让 scheduler 再释放掉这个锁。

这是让其他的核在能够看到某个线程的状态时，能够保证获取了它的锁，因此也保证其他的核没有办法获取这个线程的状态。

另外一个限制：线程除了 p->lock 以外，没有获得其他锁。这是一个非常重要的原则。

假设有一个锁 L ，进程 A 和进程 B 都能拿来使用，在进程 A 通过 switch 切换走的时候，如果进程 B 想要用 L 锁，会在这 spinning ，进程 B 没法 switch 回来让 A 继续完成自己的功能，从而导致死锁。

同协：
- 减少 spinlock ，减少忙等所带来的资源消耗。
- 为什么 sleep 要有锁的样子来做：同时让两个线程去访问一个驱动是非常危险的，此外对于部分共享数据，如用于得知信息的标志的访问，是需要用锁来做以保证安全性的。

通过 uartwrite 解释了我们上一章的 lab7 之中，为什么 Unix 针对 condition 的接口是长那个样子的。

这边课上讲了一个现象，当 release 锁的同时开启中断的时候，立刻中断就发生了，在 borken_sleep 还没有发生的时候， wakeup 就已经触发，从而没有唤醒任何线程。这个问题叫做 lost wakeup 问题

这个容易出错的窗口需要关掉，需要有一个锁保护它的睡觉状态，原语上要求，它会原子地使得进程睡眠并且释放锁，让这个行为不可分割。这样 wakeup 就不会看到对面不是处于睡眠状态的这种情况了。

实施起来，可以通过其他状态锁，保证在 sleep 获取到相关的锁之前，wakup 都没法生效。通过用两个锁覆盖了临界区域，来避免可能存在的攻击窗口。

### 如何关闭线程？
不可以简单地杀死
- 其他线程有可能在使用它的栈
- 它还有一些资源正在投入使用，尤其是在正在运行的时候，很有可能还是处于中间状态

 Unix 的哲学： wait 与 exit 协作以彻底释放全部的子进程资源。

kill 并不能真正地杀死一个进程，这是 Unix 以及 xv6 的哲学。要在后面自己慢慢回收掉。

这一节课收货挺多的，继续加油。

## Lec 14: FileSystem
文件系统的结构

inode <- file info, independent of name

注意 inode 仅仅只是一个数字节点，只有当对于 inode 的索引 link 和 open 的 fd 均为 0 时，我们才可能删掉这个 inode

### FS Layers
```
======================
 names / fds
======================
  inode -> read/write
======================
  icache -> sync
======================
    Logging
======================
  buf  cache
======================
    Disk
```
### 存储设备
一些术语
- HDD: 10msec
- SDD: 10 us - 1 ms
- Sector: 512 bytes
- blocks: 由文件系统定义，在 xv6 中是 1024 字节

CPU 通过 PCIe 设备针对 SSD 进行读写工作
### Disk layout
直接把它当做大型数组就可以了。

在 xv6 中的结构像是如下所示：
```
block 0: 空的
block 1: 超级块，用来描述文件系统
blcok 2 - 31: 日志信息
block 32 - 45: inodes - 64 byes for each inode
block 45 - 46: bitmap
block 46 ~ : data blocks (or metadata block)
```
e.g. read inode 10 => 32 + inode number(a.k.a 10) * 64 / 1024 以此得知我们从哪一个 block 中获取 inode 10 的值
### On-disk inode
inode 代表了一个文件的实体，可能对应多个文件的名称
```
type
nlink: link count to a inode
size: in bytes
blkn 0
...
blkn 11 (for direct block)
blkn 12: indirect block, point to the indirect number. 对应 256 个块，每个块 ID 的长度为 4 字节
```
文件的读写其实是遵循上面的inode结构，并以此为基础来做更新。因此inode的结构非常重要，可以解释一些读写行为为什么存在。

对于文件的data block，如果文件夹下出现了新的文件，需要在data这边做更新，以更新文件夹自身存储的内容。

因此一个最大文件大小是 268 KB
### Directories
```      
         2          14 bytes
entry inode num  filename
```
对于 root inode ，找到文件 /y/x 的流程为：首先在 root inode 对应的 inode 中，找到 y 字符串是否存在，如果存在则继续读这个 inode 。逐步向下 scan 进去。


其他一些观点：
- 采用睡眠锁是考虑了本身 buf 涉及磁盘操作，动作会比较慢，不需要忙等。并且睡眠锁确实相比自旋锁有很多性能的优势。

## Lec 15: Crash safety
Frans Kaashoek 对于 debug 的建议：
- 可以考虑最传统的 printf 大法
- 可以采用一些竞态检测器
- 可以多加上 assert 来做处理，以让程序的行为在我们的预期之中

真实的问题：崩溃真的有可能会让文件系统处于不太正确/内容不太一致的样子。

问题在于文件系统往往会多步操作，一旦 crash 出现在关键的位置，那可能文件系统就会出错，出现正确性不足的问题。

如何缓解这个问题？logging

crash may leave fs invariants violated. => After reboot, we might crash again, or we don't crash, while the R/W data is not correct.

### Solution:
logging
- atomic fscalls
- fast recovery
- high performance

不同于直接操作内存，我们会把写入的记录在日志中详细地记录下来。

步骤如下：
1. log writes
2. commit op：记录一组写入的日志信息，这一组日志中的情况一定在之后要被执行
3. install：我们完成本应完成的真正 fs 操作
4. clean log

在1/2/4被打断什么也不会发生，在3被打断我们只要重新做一遍就好了，虽然是覆盖写，但是不会影响正确性。

commit的行为是原子的，commit本质上是对某一个块进行写，在文件系统的假定中，这个是不可再往下分割的。所以就是要么不做要么做绝。

其他一些有意思的事
- begin_op 和 end_op 伴随了系统的全局，以保证我们的文件系统操作是原子的
- 
具体细节还是在看书和看代码时再慢慢落实。

### Challenges
- 如果bcache要满了，不可以驱逐bcache中用于存储log信息的块，（如果要驱逐的话，则说明要从cache直接写回disk）否则将会违反先写log再真实操作disk的规则。如果中途崩溃了，一切就完了，从而破坏了原子性
- 为了让log能够被完全应用，存放log的空间要足够大，足够大到对全部文件系统里头都做读写这种情况，依旧能够良好地记录相关日志。但更好地方式是，当有较多地请求，把他们放在一起，减少log地数目，从而提高鲁棒性
- 不可以write data from calls still in a transaction in parallel
## Lec 16: More Logging
