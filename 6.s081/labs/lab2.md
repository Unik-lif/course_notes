## 实验概览：
目的似乎是搞清楚`xv6`的一些启动流程，以及调用系统调用时的一些细节，这里涉及了一些xv6相关源代码的阅读工作，还有一些`riscv64`的细节。我们最好还是在阅读完所需代码之后，再考虑做这个实验。

## 源码解读：
### 启动：
代码段进入位置在程序`entry.S`位置处。我们可以对照`kernel.ld`文件判断，内核进入运行的第一个程序便是`_entry`位置的程序。在`_entry`之前的程序由`qemu`来做模拟。
```
	# qemu -kernel loads the kernel at 0x80000000
        # and causes each CPU to jump there.
        # kernel.ld causes the following code to
        # be placed at 0x80000000.
.section .text
_entry:
	# set up a stack for C.
        # stack0 is declared in start.c,
        # with a 4096-byte stack per CPU.
        # sp = stack0 + (hartid * 4096)
        la sp, stack0
        li a0, 1024*4
        # one hardware thread corespond to one cpu.
        # therefore we can allocate sufficient stack memory space for per CPU.
	    csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
	# jump to start() in start.c
        call start
        # never reach this line.
spin:
        j spin
```
`qemu`在启动时就已经是多核状态，这一点可以从启动脚本中看出：
```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::26000
```
我们面向的是三核`SMP`，并且通过`tcp::26000`端口为`gdb`提供了一个监听的窗口，可以在`gdb`中通过`target remote`方式进行调试。

这个程序通过`mhartid`寄存器得到当前`CPU id`信息，从而方便为不同的`CPU`设置他们所对应的栈空间，每一个栈的大小都是`4 MB`，在通过各个`CPU`核的`sp`相加之后，我们有了充足的栈空间以进行函数调用，于是通过`call start`来调用函数`start`。

`start`函数中涉及部分`RISCV`特权级机制，我们简要介绍一下：
```C
void
start()
{
  // set M Previous Privilege mode to Supervisor, for mret.
  // store machien mode status register in x.
  unsigned long x = r_mstatus();
  x &= ~MSTATUS_MPP_MASK;
  x |= MSTATUS_MPP_S;
  // therefore the MPP is set to 01, corresponds to supervisor mode.
  w_mstatus(x);

  // set M Exception Program Counter to main, for mret.
  // requires gcc -mcmodel=medany
  // similar to linux 0.11 with the way to jump back to userspace. Here we use mret to jump back to supervisor mode (since the mstatus's MPP is set to supervisor mode).
  w_mepc((uint64)main);

  // disable paging for now.
  // the 'mode' field of satp is set to zero, therefore the paging is disabled.
  w_satp(0);

  // delegate all interrupts and exceptions to supervisor mode.

  // medeleg and mideleg registers are used to delegate traps and exceptions / interrupts handling process to S-mode.
  // sedeleg and sideleg registers are very similar, but are used to delegate traps/interrupts handling process to U-mode.
  // in the specification, only 16-bit of the field got corresponding interrupts/traps info.
  
  // "To increase performance, implementations can provide individual read/write bits within medeleg
  // and mideleg to indicate that certain exceptions and interrupts should be processed directly by a
  // lower privilege level." So simply set 0xffff will be enough to delegate.

  // btw: Delegated interrupts result in the interrupt being masked at the delegator privilege level.
  // so the initialized timer won't work in Machine Mode.

  // local interrupts like timer interrupts in M-mode is hardwared hard-coded as zero, so the timer here is still be handled using 
  // m-related registers
  w_medeleg(0xffff);
  w_mideleg(0xffff);

  // SIE_SEIE: Enable S-Mode External Interrupts. => Peripheral
  // SIE_STIE: Enable S-Mode Timer    Interrupts. => Timer to let the OS get control
  // SIE_SSIE: Enable S-Mode Software Interrupts. => Software wants higher privileges
  // After all, let S-Mode handles all kinds of interrupts.
  w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // ask for clock interrupts.
  timerinit();

  // keep each CPU's hartid in its tp register, for cpuid().
  int id = r_mhartid();
  w_tp(id);

  // switch to supervisor mode and jump to main().
  asm volatile("mret");
}
```
在`riscv`中我们一开始默认状态是`M-Mode`，机器的状态存放在`mstatus`寄存器中，这里通过两个位操作，来表明之后我们希望通过`mret`进入的状态是`S-Mode`。实际上这边的用法和`Linux 0.11`第一次进入`userspace`的方法还是比较接近的，通过中断处理返回的流程来实现，并且指明了之后跳进去的函数是`main`函数。

之后，通过设置`satp`寄存器的`MODE`位，关闭页表机制。

再者，利用`medeleg`和`mideleg`寄存器实现`exception`和`mideleg`的降权处理，每一个`bit`对应一个中断类型，让本来在`M-Mode`下处理的程序转交给`S-Mode`来做。然而，即便采用了降权也不完全意味着我们全部的中断都被下放到`S-Mode`了。在`M-Mode`中存在着时钟中断和外部中断，至少在`xv6`默认采用的`SFive`对应硬件中，它们所对应的`medeleg`和`mideleg`位置是被硬编码为`0`的，也就是说这边直接通过设置`0xffff`的方式倒不如是为了方便起见。

之后，需要`w_sie`来实现`S-Mode`下部分`interrupt enable`设置，以实装`S-Mode handles all kinds of interrupts`一说。

之后，我们通过`timerinit`函数来对计时器进行初始化，这个函数我们等会儿做分析。

通过下面的设置，将`thread`号读入到`tp`之中，以符合`riscv`的规范。完事儿之后，再通过`mret`从`M-Mode`通过中断恢复跳转回`S-Mode`，将控制流交送给`main`函数。
```C
  int id = r_mhartid();
  w_tp(id);
```

在进入`main`函数之前，我们再来看看`timerinit`。该函数主要与硬件组分`CLINT`进行交互，`CLINT`组件一般负责`Qemu`时钟周期的控制，通过`timecmp`机制来进行触发。`CLINT_MTIME`会随着时间逐渐增长，而`CLINT_MTIMECMP`则是用于与这个增长参数进行比较的参数。

特别的，其会触发`M-Mode`下的中断，在`RISCV`手册中提到，处理`x-Mode`下中断不能是比`x`更低的权限级。下放中断处理权只是给予`x-Mode`下中断触发时无需陷入到`M-Mode`这一默认行为的权利。因此，在后续还是需要对`M-Mode`相关的中断处理函数做简单的设置。
```C
// set up to receive timer interrupts in machine mode,
// which arrive at timervec in kernelvec.S,
// which turns them into software interrupts for
// devintr() in trap.c.
void
timerinit()
{
  // each CPU has a separate source of timer interrupts.
  int id = r_mhartid();

  // ask the CLINT for a timer interrupt.
  // This kinda info is hard to search. especially for the cpu_id and the CLINT_MTIMECMP.
  // Nah.. Lets just assume this is right ...
  int interval = 1000000; // cycles; about 1/10th second in qemu.
  *(uint64*)CLINT_MTIMECMP(id) = *(uint64*)CLINT_MTIME + interval;

  // prepare information in scratch[] for timervec.
  // scratch[0..3] : space for timervec to save registers.
  // scratch[4] : address of CLINT MTIMECMP register.
  // scratch[5] : desired interval (in cycles) between timer interrupts.
  // use scratch this very handy register to store sth, therefore when trap happens, we could easily fetch 
  // info from scratch register.

  // mscratch: a register that every cpu has a local one. (hart-local)
  // cpu 0 info | cpu 1 info | .... | cpu 8 info |
  uint64 *scratch = &mscratch0[32 * id];
  scratch[4] = CLINT_MTIMECMP(id);
  scratch[5] = interval;
  w_mscratch((uint64)scratch);

  // set the machine-mode trap handler.
  // when mtvec got the MODE as direct, all traps into machine mode cause pc to point to timervec.
  w_mtvec((uint64)timervec);

  // enable machine-mode interrupts.
  // first step.
  w_mstatus(r_mstatus() | MSTATUS_MIE);

  // enable machine-mode timer interrupts.
  // when MIE is set, then we can set this kinda interrupt.
  w_mie(r_mie() | MIE_MTIE);
}
```
这边设置了`interval`为`0.1s`，即每经过`0.1s`，时钟中断将会被触发，以让内核重新掌控机器。这边会进入到`timervec`这个函数。

其实这边的行为似乎还是比较简单，首先这边设置了一个`.align 4`的对齐操作，这个对齐与`mtvec`的设置有关，能够保证最后两位`bit`所对应的`Mode`位置为`0`，从而设置其为`Direct`状态。

这边主要对于先前存放好的`mscratch`来进行操作，每个`CPU`核都能用到他们所对应的那一组数据。操作的流程是很经典的存放-操作-恢复的流程。
```S
.globl timervec
.align 4
timervec:
        # start.c has set up the memory that mscratch points to:
        # scratch[0,8,16] : register save area.
        # temporary save a1, a2, a3 in scratch
        # scratch[32] : address of CLINT's MTIMECMP register.
        # scratch[40] : desired interval between interrupts.
        
        csrrw a0, mscratch, a0
        sd a1, 0(a0)
        sd a2, 8(a0)
        sd a3, 16(a0)

        # schedule the next timer interrupt
        # by adding interval to mtimecmp.
        ld a1, 32(a0) # CLINT_MTIMECMP(hart)
        ld a2, 40(a0) # interval
        ld a3, 0(a1)
        # add interval to a1, get the new CLINT_MTIMECMP
        add a3, a3, a2
        sd a3, 0(a1)

        # raise a supervisor software interrupt.
        # the second phase of time trap.
	    li a1, 2
        csrw sip, a1

        ld a3, 16(a0)
        ld a2, 8(a0)
        ld a1, 0(a0)
        csrrw a0, mscratch, a0

        mret

```
值得注意的是在这边通过`csrw sip, a1`进行了`S-Mode`下软中断的触发，这边是系统调用的活，我们暂时不用做。因此时钟中断的流程可以分为`M-Mode`下中断处理和`S-Mode`下的软中断处理一共两个阶段。

好的，回到`main`函数这边，这边设置了一个系统启动的基础框架。
```C
// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  // bsp: 0, others are aps. should wait until the bsp has lanuched.
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode cache
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```
我们`qemu`启动的时候是用了三个`CPU`核，因此是存在一个先来后到关系，为了实现这一点，此处设计了一个不会被编译器所优化的变量：
```C
volatile static int started = 0;
```
来满足这一点。

`thread 0`做完一些基础工作后，才能够轮到其他线程来做事情。我们大意看到这样就差不多了，现在关键问题是`thread 0`，即普遍意义上的`BSP`做了什么？

本节我们只关心启动，我们进入到`userinit`函数简单分析一下，并且知道个大概就够了，因为还没轮到去分析内存分配等部分。
```C
// a user program that calls exec("/init")
// od -t xC initcode
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x45, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x35, 0x02,
  0x93, 0x08, 0x70, 0x00, 0x73, 0x00, 0x00, 0x00,
  0x93, 0x08, 0x20, 0x00, 0x73, 0x00, 0x00, 0x00,
  0xef, 0xf0, 0x9f, 0xff, 0x2f, 0x69, 0x6e, 0x69,
  0x74, 0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00, 0x00
};

// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  // initcode here is the string form of file initcode.asm
  // take a look at this file.
  // initcode calls the SYS_EXEC syscall at the end => trap context switch process is involved here.
  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```
我们的分析在这边仅局限到注释层面，更深的层次我们就不再考虑了。首先，在进入`userinit`函数之后，我们起了一个`proc`即进程类型来进行管理。

在文件`proc.h`中我们看到进程类型的结构，该结构管理进程的一些状态，包括父进程与退出状态等，此外还有进程所对应的上下文信息、页表信息、文件描述符信息等等。
```C
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```
对于进程的分配，此处采用`allocproc`来做，我们需要注意到`xv6`对于进程数目的总量有一定的限制，最大是`64`个，初始化时还有对部分上下文状态做初始化，此处便按下不表。

之后，利用`uvminit`函数把字符串`initcode`中的内容拷贝到进程所对应的地址空间之中。其他的操作看看就得了，不难理解。

需要注意的是，`initcode`的具体信息如下：
```S
# Initial process that execs /init.
# This code runs in user space.

#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0
```
通过系统调用，`SYS_exec`，进入到系统调用的`trap`位置处。处理完之后，我们系统的初始化便完成了。不过还需要细化一点，系统调用的流程和系统调用时怎么初始化的，为此我们需要看一下先前的部分函数。
### 内核中断
我们尝试分析`trapinit`函数和`trapinithart`函数。
```C
void
trapinit(void)
{
  initlock(&tickslock, "time");
}
```
该函数似乎没做啥，对于某个自旋锁做了初始化操作。

之后用`trapinithart`函数设置当前`CPU`核所对应的`stvec`寄存器，让其trap发生之时，能够指向`kernelvec`所对应的位置，这边是我们处理中断的关键函数入口，对此，我们需要研究这个位置的函数情况。

说起来这个函数似乎也比较简单，把全部的通用寄存器存了一下，一共有`32`个寄存器，每个对应八字节，所以一开始`sp`要减少`256`，在利用kerneltrap函数处理完中断处理函数后，我们再从先前的`sp`恢复上下文信息。
```S
	#
        # interrupts and exceptions while in supervisor
        # mode come here.
        #
        # push all registers, call kerneltrap(), restore, return.
        #
.globl kerneltrap
.globl kernelvec
.align 4
kernelvec:
        // make room to save registers.
        addi sp, sp, -256

        // save the registers.
        sd ra, 0(sp)
        sd sp, 8(sp)
        sd gp, 16(sp)
        sd tp, 24(sp)
        sd t0, 32(sp)
        sd t1, 40(sp)
        sd t2, 48(sp)
        sd s0, 56(sp)
        sd s1, 64(sp)
        sd a0, 72(sp)
        sd a1, 80(sp)
        sd a2, 88(sp)
        sd a3, 96(sp)
        sd a4, 104(sp)
        sd a5, 112(sp)
        sd a6, 120(sp)
        sd a7, 128(sp)
        sd s2, 136(sp)
        sd s3, 144(sp)
        sd s4, 152(sp)
        sd s5, 160(sp)
        sd s6, 168(sp)
        sd s7, 176(sp)
        sd s8, 184(sp)
        sd s9, 192(sp)
        sd s10, 200(sp)
        sd s11, 208(sp)
        sd t3, 216(sp)
        sd t4, 224(sp)
        sd t5, 232(sp)
        sd t6, 240(sp)

	// call the C trap handler in trap.c
        call kerneltrap

        // restore registers.
        ld ra, 0(sp)
        ld sp, 8(sp)
        ld gp, 16(sp)
        // not this, in case we moved CPUs: ld tp, 24(sp)
        ld t0, 32(sp)
        ld t1, 40(sp)
        ld t2, 48(sp)
        ld s0, 56(sp)
        ld s1, 64(sp)
        ld a0, 72(sp)
        ld a1, 80(sp)
        ld a2, 88(sp)
        ld a3, 96(sp)
        ld a4, 104(sp)
        ld a5, 112(sp)
        ld a6, 120(sp)
        ld a7, 128(sp)
        ld s2, 136(sp)
        ld s3, 144(sp)
        ld s4, 152(sp)
        ld s5, 160(sp)
        ld s6, 168(sp)
        ld s7, 176(sp)
        ld s8, 184(sp)
        ld s9, 192(sp)
        ld s10, 200(sp)
        ld s11, 208(sp)
        ld t3, 216(sp)
        ld t4, 224(sp)
        ld t5, 232(sp)
        ld t6, 240(sp)

        addi sp, sp, 256

        // return to whatever we were doing in the kernel.
        sret
```
下面分析函数`kerneltrap`，其信息如下所示：
```C
// interrupts and exceptions from kernel code go here via kernelvec,
// on whatever the current kernel stack is.
void 
kerneltrap()
{
  int which_dev = 0;
  // address, therefore we can jump back?
  uint64 sepc = r_sepc();
  // sstatus of the S-Mode when trap happens.
  uint64 sstatus = r_sstatus();
  // casue will be stored here.
  uint64 scause = r_scause();
  
  if((sstatus & SSTATUS_SPP) == 0)
    panic("kerneltrap: not from supervisor mode");
  // SIE should be set to enable interrupts.
  if(intr_get() != 0)
    panic("kerneltrap: interrupts enabled");

  if((which_dev = devintr()) == 0){
    printf("scause %p\n", scause);
    printf("sepc=%p stval=%p\n", r_sepc(), r_stval());
    panic("kerneltrap");
  }

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2 && myproc() != 0 && myproc()->state == RUNNING)
    yield();

  // the yield() may have caused some traps to occur,
  // so restore trap registers for use by kernelvec.S's sepc instruction.
  w_sepc(sepc);
  w_sstatus(sstatus);
}
```
需要注意的是，既然我们的`trap`已经被下放到了`S-Mode`，就会有一些特殊的寄存器设置：
1. `scause`寄存器中会写入`trap`的原因
2. `sepc`寄存器中会写入发生`trap`的指令虚拟地址
3. `stval`寄存器中会写入一个`exception-specific`数值
其他的细节在这里展示：
```
the SPP field of mstatus is written with the active privilege mode at the time of the trap; the SPIE field of mstatus is written with the value of the SIE field at the time of the trap; and the SIE field of mstatus is cleared. The mcause, mepc, and mtval registers and the MPP and MPIE fields of mstatus are not written.
```
前面的函数似乎都不太麻烦理解，重点看向函数`devintr`，这个函数如下所示：
```C
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  uint64 scause = r_scause();

  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000001L){
    // software interrupt from a machine-mode timer interrupt,
    // forwarded by timervec in kernelvec.S.

    if(cpuid() == 0){
      clockintr();
    }
    
    // acknowledge the software interrupt by clearing
    // the SSIP bit in sip.
    w_sip(r_sip() & ~2);

    return 2;
  } else {
    return 0;
  }
}
```
这边会根据`scause`中的信息来判断我们的内核中断究竟是由谁来触发的，具体的一些硬件细节可以查看手册，这边我不怎么想看了，感觉似乎没有太大意思，还是省一点力气吧。
### 系统调用
我在原`main.c`中找了一段时间，一开始并没有找到`usertrap`相关的设置，后来意识到这个东西可能和进入的`userspace`函数`userinit`函数有一些关联。

我们重新看向函数`userinit`，仔细理解一下。注意到这边的`allocproc`内的情形：
```C
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```
该函数首先检查在`proc`数组中是否有`Unused`的项，如果有，那么进入`found`，表示找到了可以添加新进程的空间。之后，为这个进程分配`pid`和`trapframe`，以及一个含有`trampoline pages`映射的页表。

最后，设置`p->context`的上下文中`ra`寄存器为`forkret`，`sp`为内核栈之上的某个位置。

现在我们暂时还不需要管调度上的事（如果还按照先前这样子来做，可能会花太多的时间），如果按照注释上的来，我们大概可以知道在调度到它的时候，会进入到`forkret`函数的入口，此时还是`S`状态，之后的程序的行为值得注意，会最后进入到`user`状态。
```C
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  usertrapret();
}
```
诶嘿，好像找到了。我们看到了`usertrapret`，钻进去玩玩：
```C
//
// return to user space
//
void
usertrapret(void)
{
  struct proc *p = myproc();

  // we're about to switch the destination of traps from
  // kerneltrap() to usertrap(), so turn off interrupts until
  // we're back in user space, where usertrap() is correct.
  intr_off();

  // send syscalls, interrupts, and exceptions to trampoline.S
  w_stvec(TRAMPOLINE + (uservec - trampoline));

  // set up trapframe values that uservec will need when
  // the process next re-enters the kernel.
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  // NOTE: Link.
  // store the satp info into the variables, therefore we can use it by passing it to fn.
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
}
```
在这边设置了`w_stvec`为`TRAMPOLINE + (uservec - trampoline)`，并且在为跳转到`userspace`之前做了就`trapframe`上的准备，诸如`kernel satp`页表位置，`kernel_sp`等等。

之后，通过使用`fn`方式来解析内核`TRAMPOLINE + (userret - trampoline)`这个地址，并且跳转到这个函数为止，在内核地址空间中恰好对应了`userret`这个函数，内核地址空间的设置在这一节暂时还没涉及（是下一集的内容啊喂），目前只要知道那个位置确实是这个函数就行了。

我们跳转过来了，来看看发生了什么事情？首先我们来看看传输进来的两个参数，一个是`TRAPFRAME`，一个是`satp`，根据`RISC-V`的调用规范，它们分别被存放到`a0`与`a1`寄存器之中。
```S
.globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        
        # NOTE: link
        # To enable the user page table to find the instructions,
        # the trampoline code should be equal in both user space and kernel space.
        csrw satp, a1
        # NOTE: link
        # clear the TLB.
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.

        # NOTE: link
        # store the a0 in the trapframe to sscratch
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	      # restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```
感觉没什么好说的，把`trapframe`全部都`load`到这些通用寄存器上面，把某个进程的上下文信息装载进来。其他的一些小细节本人在这边用`# NOTE`注释做了标注。完成了这些事情后，我们会通过`sret`从`S-Mode`进入`U-Mode`，跳转回的地址是`sepc`寄存器所指示的位置。而在一开始的`userinit`中，这个地址指向了用户程序的`0`虚拟地址，这就有意思了。

先前我们在上一节看到，为`0`的虚拟地址被存放了`initcode`中的字节信息，于是`initcode`中的指令就跑起来了，紧接着就开始进行一些系统调用的事情。特别的，我们注意到在上面分析的过程中，系统调用相关寄存器的设置已经完成了，现在我们跟踪一下`initcode`中的一些代码。
```S
# Initial process that execs /init.
# This code runs in user space.

#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = "/init\0";
init:
  .string "/init\0"

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0
```
在这边通过`ecall`进入系统调用窗口之前，于此同时设置了`a7`为`SYS_exec`，设置了`a0`为`init`，`a7`为`argv`。根据先前的设置结果，`ecall`会进入`trampoline`中的`uservec`函数，方位由`stvec`寄存器所设置。特别的，我们需要注意，上一次进入离开内核态之时，`sscratch`寄存器已经被设置成了`trapframe`地址。
```S
	      #
        # code to switch between user and kernel space.
        #
        # this code is mapped at the same virtual address
        # (TRAMPOLINE) in user and kernel space so that
        # it continues to work when it switches page tables.
	      #
	      # kernel.ld causes this to be aligned
        # to a page boundary.
        #
.section trampsec
.globl trampoline
trampoline:
.align 4
.globl uservec
uservec:    
      	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #
        
	      # swap a0 and sscratch
        # so that a0 is TRAPFRAME

        # NOTE: link
        # a0: init
        # a1: argv
        # a7: SYS_exec

        # sscratch always 
        csrrw a0, sscratch, a0

        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

        # kernel stack should set afterwards.

      	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        # restore kernel stack pointer from p->trapframe->kernel_sp
        
        # NOIE: kernel stack should be set 
        # or we can't call functions in kernel mode.
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), p->trapframe->kernel_trap
        ld t0, 16(a0)

        # restore kernel page table from p->trapframe->kernel_satp
        
        # NOTE: link
        # set page table back to kernel space. / set satp back to kernel space. 
        ld t1, 0(a0)
        csrw satp, t1
        sfence.vma zero, zero

        # a0 is no longer valid, since the kernel page
        # table does not specially map p->tf.

        # jump to usertrap(), which does not return
        jr t0

```
很自然，最后控制流跳转到了`usertrap`位置处。首先检查是否是从`user mode`跳转过来的，这个`SPP`表示`smode previous privilege`，它理应是`0`，所以会有这样的一个判别。

由于现在我们的状态在`S-Mode`之中，中断理应都来自内核自身，所以特地做了这样一个设置`kernelvec`。
```C
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.

    // NOTE: link
    // at least our settings are now completed. we can now set the interrupts on.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```
中断发生时，`sepc`寄存器自动存放着该跳转回去的`pc`地址，`scause`寄存器中存放着原因。在`syscall`的情况下，我们简单做一些设置，便通过`syscall`函数进入到系统调用`handler`位置了。现在我们便可以分析`syscall`函数。
```C
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // call the syscalls[num]() function, return the value to a0 register in trapframe.
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
感觉这一步非常自然，把`trapframe`中的东西弹出来。原来一个较好版本的`syscall`信息是这么传递的，感觉确实要比`linux-svsm`中因为机密虚拟机而直接来做的方式要漂亮很多，也不需要那么`tricky`了。总之根据先前`a7`的值确定系统调用的号，调用这个函数，并且把结果返回到`a0`寄存器之中，这件事情就算是做完了。

我们不具体来看在启动阶段调用的系统调用`exec`这个函数做了什么了，简单一看，确实该干嘛就干嘛嘛，像`fork`这一类的函数往往会比较麻烦，需要把全部机制搭建好再跑`exec`指令的。仅仅通过这么一则推文就尝试把全部细节搞清楚，是不是有点耍赖了呀。

完成了`syscall`函数后，简单做一下判别（对应`else if`中的设置情况），决定是否通过`yield`来挂起`CPU`。

最终，调用`usertrapret`函数，但是这一次的`epc`和`myproc`就得听`scheduler`和用户进程本身的话了。我们暂且不表。

那么到这里，我们的源码解读就结束了。`risc-v`的细节虽然没有`x86`一样多，但是它的手册写的不是很好，想把整个事情这样梳理下来，我感觉还是有点点费劲的，更不用说一开始去写`xv6`内核的人了，感觉他们真的是很厉害的程序员。

需要特别注意的是，第一个系统调用的过程我们可以清楚看到，那接下来的应用程序呢？

其实原理非常简单，`usys.S`生成的代码其实就是`user.h`中那么多系统调用的具体实现，你调它的时候其实就会跑`ecall`然后就跑到内核去了。

## 实验：
目前手头有个项目在做，打算业余抽一点点小时间来做（实际上这边的源码解读也是业余做的呜呜呜）

花了三个小时做完了这个实验，这个实验非常简单，在我们搞清楚了系统调用的流程之后，感觉完全没怎么费力就做完了，很轻松。

哎，看来自己还是有那么一点点进步的吧。

唯一需要注意的是`mask`可以添加在`proc`之中，以前做`rCore`的时候不太清楚类似的东西存在哪里，现在感觉存在`proc`结构体中是非常自然的事情。

感觉一共两个实验写了`100`行代码都没有，至少这个实验可能就三十多行，上一个实验的代码量稍微大一点，我写了`8`小时。
## 实验总结：
感觉还是练手，主要的提高实在读源码部分，和`rCore`风格如出一辙，或者说`rCore`方法论学的很好。