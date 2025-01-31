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

特别的，log的结构是，首先有一个log header，用来存放每一个log块之后要写到的位置，记录block number，然后紧紧跟着这些块所对应的数据信息。

直到write_head结束之后，我们才能恢复崩溃的数据。
### Challenges
- 如果bcache要满了，不可以驱逐bcache中用于存储log信息的块。否则，相比于其他log，这个log对应的信息事先更新到disk里了，破坏了原子性。正常情况下，log应该是一起更新到disk里头的。
- 为了让log能够被完全应用，存放log的空间要足够大，足够大到对全部文件系统里头都做读写这种情况，依旧能够良好地记录相关日志。然而考虑到确实有写大文件的需求，这种情况下需要保证每次写的大小不超过transcation可以记录的大小，先做拆分，但最后还是把这些拆分成比较小的行为视作一个整体的行为。也就是最后也还是算是一次操作。
- 如果要concurently写transcation，必须要保证能够fit进去，如果空间不够，没有办法一口气完成一个事务，那就会破坏原子性。因此，需要限制同时间的fs calls总数目。我们或许可以让它保证先把一个fs calls对应的log写完后，再尝试放第二个fs calls，但这并不是唯一的设计。在xv6中，我们确实可以同时按照不同的顺序来写，甚至交替着写，但要保证所有出现的task都能彻底地完成，也就是彻底地fit进去，因此要做一些基本的检查。


在transcation中，对于系统调用做排序是很重要的。
## Lec 16: More Logging
log = journal, ext3 = ext2 + journal

xv6 log review

为什么linux不用xv6相同的方案？主要原因还是，xv6的logging方案太慢了，每一个syscall都要经过一个完整的拷贝到logging block中的过程，耗时巨大。
### Ext 3
实际上对于底层的EXT 2系统做的改动非常小。 
 
Ext3将会维持transcation info信息：
1. 事物修改的序列信息，以及block号
2. handles

在磁盘的特定位置，同样维持一个LOG信息。EXT3的特性在于，其能够同时追踪多个事务。

基本的结构superblock：
- 首先有一个log superblock中包含superblock内的第一个有效事务的偏移量和对应的序列号
- 之后的每一个transacation中都包含着一些描述块
- 最后，还可能有多个commit block作为完结数据

以一个环形链表的方式来组织
### 性能提高手段
1. 系统调用只是更新内存缓存块中的内容，便可以返回，并不需要经过IO操作
2. Batching操作
3. Concurrency

#### 系统调用的异步性：
只是修改缓存中的块，并没有涉及磁盘上的写。这让我们可以在后续实现IO的concurrency，可以避免程序执行和IO操作的重叠。此外，这也让batching操作变得更加容易。但是，即便系统调用成功返回了，这也不意味着我们的系统调用真的落实下来了。

为了避免上面的问题，现在有了一个不错的程序接口，叫做fsync，可以让系统调用对于文件的操作能够真实的同步到磁盘中，和flush很像。
#### Batching的好处
在EXT3中总是有一个open transcation事务，在这之后，系统调用们他们写入的都是这个大的transcation的一部分，这摊销了很多需要多次在机器驱动上写的成本。同时，和xv6的系统一样，batching也支持我们absorbtion，可以明显减少我们写入的次数。

此外，还有一个好处是disk scheduling。如果有大批量的写的话，可以让磁盘的利用率更加高，能够更早地完成应该完成的任务。
#### Concurrency
1. 允许多个syscall的并行。彼此之间并不需要有依赖时间关系。因为修改的都是一个单独的一个区域。
2. 很多老的事情都可以同时并行存在，它们可能都是存在于不同的生命周期中。

### 真实代码：
在start的时候，提供一个句柄h，作为一个事务作用的单元。然后，对于块操作，需要把h和这些blocks的关系绑定在一起。

在commit的步骤如下：
1. block new system calls
2. wait for outstanding syscalls (write to cache)
3. open new transcation
4. write description block and related data  blocks
5. write blocks to log
6. wait for finishing
7. write commit block
8. wait for commit write finish (reach the 'commit point', guarantee the modification is now safe)
9. write to home locations (a little long)
10. re-use log
### 恢复
每个transacation会留下来description block，留一个magic number之类的值，用来让恢复软件知道，崩溃是从哪一个事件位置开始的。

但是在xv6中，并没有这个多个transcation的机制，只有一个事务。因此并发性在xv6中是不适用的。

### 其他细节
EXT3要求一个事务T1的所有的syscalls被完成后，另外一个新的事务T2的syscalls才可以跑起来。

这边举了一个例子，大概是在一个Task的commit还没有完成的时候崩溃了，此时得到的结果就是，两个文件使用同一个inode。

这边MIT的学生提了一些问题，非常好，其中老师解释了一种情况，也就是首先T8这个事件上来之后，要跑了前几个block之后，系统才能知道事务所需要的blocks具体的数目。因此，确实还是存在着覆盖了后面的T5的可能性。

## Lec 17: VM for application
希望让用户程序与内核有相同的机制，应用程序可以访问用户应用程序，让应用程序可以响应这些页面错误。可以修改页面的保护位。

在这篇论文里，提到仅需要使用一小部分的虚拟内存原语，就可以得到很多类型的应用程序。

基本的原语：
- TRAP: 响应来自页面的错误 - sigaction
- Prot1: decrease accessibility of One Page                                        - mprotect 
- ProtN: decrease accessibility of N Pages, 与N次prot1相比，可以减少TLB刷新的次数     - mprotect
- Unprot: increase accessibility                                                   - mprotect
- Dirty: return a list of dirtied pages since the previous call
- Map2: map the same physical page at two different virtual addresses. - mmaps

### Unix today: mmap, mprotect
```C
addr = mmap(null, len, R/W, MAP_PRIVATE, fd, offset);
mprotect(addr, len, Read_ONLY) -> lds
sigaction: signal handler of the user application
```
### Implementation of VM
Pagetable + Virutal Memory Area (VMAs) => Contiguous range of addresses, same permission baced by same object

### User-level traps:
PTE marked invalid -> CPU jumps to kernel -> kernel save states -> ask the VM system what to do -> upcall into user space -> run handler -> mprotect()? -> handler returns to kernel -> kernel resumes interupted process.

在这边通过upcall into user space，本质上与触发问题的user program拥有相同的程序上下文，这自然也是一个利好之一。

### Examples
#### a huge memorization table
the table store expensive results of function => pre-computed

Challenge: the table might be huge, bigger than physical memory.

Solution: use VM primitives.
- allocate huge range
- table[i] -> page fault
- -> allocate page -> compute f(i) -> store in table[i]

if much memory is in use, free some of the entries.
#### Garbage Collector
a copying garbage collector

将内存分割成两个部分

从root开始，遍历所有的object，把from空间的所有的能够指向的object都拷贝到to空间中。

拷贝的同时做扫描，尽可能地拷贝所有live状态的object指针

这个过程被称为forwarding，持续复制过来更新整个object的环形结构，forward全部的对象，然后把整个from空间都删除。

这样，处于dangling和idle状态的东西自然而然就全部丢掉了。

该算法被称为baker's算法，是实时并且incremental的一种算法。

麻烦的地方：
1. incremental地解引用，很麻烦
2. 让这个东西并行其实并不容易，存在一定的风险

论文描述的更新方案：如果我们有用户级别的原语，可以几乎免费地获得并发性。

使用fault handler流程，事先把to部分全部用page map方式设置成NONE，scan one page of objects + forward

在完成了这一次handler之后，就把这个区域重新加上unprotect机制，把单元一个一个地送到scan部分。

更好的地方是不需要再做检查指针了，可以通过虚拟化的硬件机制，也就是sig handler机制来实现这一点了。

这边的操作被简化为copy一页，然后通过fault handler来实现功能。unscan部分被设置为None的page bit.

#### Tricky
使用map2映射物理内存两次，来操作并没有被分配的页面内容。

### 总结
这篇paper我有点没看懂，先全部通读一遍，之后再考虑其他细节。             

## Lec 18: Microkernels
tranditional kernel: monolithic

big abstraction - e.g. FileSystem
- portability
- hide complexitiy
- resource management

These abstraction make the kernel very complex and big

The Isolation of different components is lacking.

Why not monolithic?
- HUGE -> Complex -> Bugs -> Security
- General purpose -> slow
- Design Baked in big Kernels
- Extensibility is not good

Microkernels Idea:
- tiny kernel support IPC, Tasks
- Many functions are implemented in User Space, and use IPC to cooperate together.

Example: L4, OpenHarmony

Why use Microkernel?
- small -> secore
- small -> verifiable - seL4.
- small -> fast
- small -> more flexible
- user level -> modular
- user level -> easier to customize
- user level -> more robust
- O/S personalities

challenges:
- minimum syscall api?
- - support exec(), fork()
- rest of OS
- - great pressure to make the IPC quick, IPC should be made quick enough

L4:
- 7 syscalls
- 13000 LoC 
- syscalls: create thread, send/recv, IPC, mappings, dev access, interrupt->IPC. yield syscalls
- Pager to handle Lazy Allocation related page Faults.

IPC slowness:
- Process1: Send(ID, MSG): append message in the buffer, receive the info to inform finishment
- Process2: Recv() -> ID, MSG from the shared buffer, later send the info to inform finishment

This is asychronous buffered method, which is pretty slow.

Another paper: L4 fast IPC (synchronous)
- Let Process 1 there is a waiting Process out here. Unbuffered.
- P1 and P2 both wait for the pointer to be valid and ready for receiving message.

Huge message can be passed in this way. This is called RPC:
- call() - send + recv
- sendrecv() - send, but wait for reciving next message

20x speed up.  

这边的模型，是用了client-server模型，有一个一直都在忙等其他人的服务。最关键的是，P1可以直接跳转到P2的上下文中，这个方式相对来说比较hacky，但是很快。

L4在userspace跑一个Linux Kernel，把Linux内部的应用程序放在外头跑，通过IPC的方式来告知L4系统做一些特权类的修改等等。

对于linux和外头的VI进程，他们各自通过一个fork来专门负责进行IPC的交互和同步。

一些时代局限性：
- Linux内部的内核进程没必要由L4来进行接管，因为当时跑单核的机器，这并不会带了更好的性能提升。
- 在当时的Linux中，Linux本身也假设一个kernel就只能跑一个核而已，即uniprocessor，没有spinlock等设计

Takeaway:
- Interesting design
- argument whether microkernel is worth using, against Mach (bad performance)
- still twice cost of the Linux (twice system calls in the process)

AIM: benchmarks using many syscalls.

这里morris还讲了一些关于生态位的思考。

在L4中一直都在不同的组件中切换，本身就在切换不同的pagetable，这一点在他们从内核态和用户态中切换中，哪怕实现了在同一pagetable中存在，也不会对性能带来很多的帮助，因为L4就始终都是在不停地切换页表的。

更加致命的一点，在微内核中为每个内核进程这么做，实在是太复杂了，根本没有什么利好，速度还很慢。

实际上微内核的方案确实在很多情况下被拿去混合着了。论文里还有一些帮助加快切换页表的小技巧。

## Lec 19: Virtual Machines
Virtual Machine: simulation of the Machine

### How to implement VM
- Guest should not break in its isolated part.
- emulation of the simulated virtual machine system.

VMM is the kernel that boots on this piece of hardware. VMM is in the supervisor mode.

VMM is a part of the Linux of the Host machine. Always VMM is a loadable part, like KVM.

Tricks:
- Run guest kernel in user mode
- when the guest os uses privileged instruction, trap to RISCV hardware
- VMM will intercept this and emulate the privileged instruction for the Guest Machine
- VMM will keep each of the Guest Machine a set of states in the memory. (VMCS or VMSA).
- All these changes to the registers are virtual, not the real registers, or it will trap into the real host os.

即便是guest OS中通过sret跳转到user mode，VMM中也会通过mode一项来做修改，因此这件事需要trap到VMM中去。

### Page Translation
Guest Page TBL: gva -> gpa.

VMM Map: gpa -> hpa.

VMM "shadow" pg tbl: gva -> hpa. this what the real hardware uses.

因此，用户需要某个gva，他就会自动翻译到hpa上。如果出现一些页面异常，会让host机器通过stap装上这个具体的shadow pg tbl，来实现地址翻译。

这目前还是没有EPT出现的时候的方案。属于trap-and-emulate的方案。

### Devices in VM
1. emulation - 原生类型机器，不需要对OS做修改
2. provide virtual devices: xv6 also uses, virtio_disk.c，这种类型的机器能够知道自己其实不是在真机上跑，类似于Xen这种类型，需要有一些针对性，可能对硬件也会有一些特定的需求
3. pass through NIC: 直连类型的设备

### Hardware Support For VM
Intel VT-x

THE VMM run in VMX 'root' mode, while the Guest Machine runs in VMX 'non-root' mode.

Two sets of privileged registers.

Another part: pg tbl.

Use EPT (extended page table), to directly use the VM's pg tbl root address without trapping.

Every core has its own set of registers.
### Dune
takes the VT-x hardware as the starting point.使用VT-x来创建一个DUne process进程，本来这是给VM来做的。

- sandbox: let the supervisor manage the cr3 of the user mode in the Dune VM.
- GC: 直接通过VT-x的cr3页面指向的页表项，根据Dirty位来判断是否是垃圾，相比原来的遍历方式，显著加快了垃圾回收的速度。

特别的，GC是一个one-pass过程，之后将会冻结一切，其他之后的修改，我就不管了。