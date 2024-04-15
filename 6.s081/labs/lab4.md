## 实验概览：
总之尝试搞清楚很多很多东西:
1. 关于`trampoline`的设置。
```shell
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
	.section trampsec # 在 kernel.ld 中标注了这一点，将这部分代码放到恰当的位置
.globl trampoline # 在后面做 mapping 的时候调用它来作为物理地址位置空间
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
```
在allocproc的时候，即userinit函数的时候就已经做好了这件事情。映射到了恰当的位置。
2. `userinit`之后，控制流是什么样子的？
首先进入scheduler函数，我们简单查看一下：
```C
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    // 让处于S权限的硬件可以随时中断程序
    intr_on();
    
    int found = 0;
    // 遍历全部的进程
    for(p = proc; p < &proc[NPROC]; p++) {
      // 获得进程所对应的锁，防止被其他核所夺取
      acquire(&p->lock);
      // 检查进程状态是否是可以跑的，即 ready 状态
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        // 让当前的 cpu 所指向的进程正好是这个 p
        c->proc = p;
        // 切换当前 cpu 的上下文状态为 p 进程所对应的上下文状态
        // &c->context 对应 a0 寄存器
        // &p->context 对应 a1 寄存器
        // 似乎 switch 的时候尚还没有像 syscall 那样正式，不需要把上下文存放到特定的位置，直接放到 context 之中
        // 跑起来这个应用程序，不过 ret 跳转到的位置，一开始还是 forkret
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```
下面是`swtch`函数的详细解释：存放上下文信息到 c->context 之中，并且用 p->context 来替换，之后正常返回就行。

特别的，这边使用了 ret 指令，那么就会跳转到 p->context->ra 所指示的位置上，那么可以换言之，我们的程序跑起来了。记得在每个进程初始化的时候，`ra`存放的东西似乎都是`forkret`函数的地址。
```C
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.	


.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```
我们进入到 forkret 函数中，我们只做必要的分析，比如 fsinit 函数我们还没用到，那就不讲了。
```C
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  // 进程到这边已经算是跑起来了，之后他自己就会很听话往下走， process 可以自己释放掉先前的进程锁， scheduler 可以重新开始调用这个应用程序了？
  // 大概？总之还是我不懂的范畴，希望之后讲课的时候能讲到。
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
进入`usertrapret`函数中查看。不过这边感觉更像是代码层面上的复用。
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
  // 让硬件没法再中断流程了，我们要进入到用户态了。
  intr_off();

  // send syscalls, interrupts, and exceptions to trampoline.S
  // 当 syscalls interrupts exceptions 发生时，跳转到 trampoline.S 所对应的位置来运行
  w_stvec(TRAMPOLINE + (uservec - trampoline));

  // set up trapframe values that uservec will need when
  // the process next re-enters the kernel.
  // 在 trapframe 中还记录了 kernel 所对应的 page table 的根目录地址
  // 不过现在的 trapframe 还没有存到 trampoline 中所对应的地址上，只是暂时存放在 p->trapframe 上而已（其实并不是）
  //
  // 检查了一下代码，发现 p->trapframe 是正儿八经通过 kalloc 来为每个进程来做分配的，并且 p->trapframe 是老老实实地写在了
  // TRAPFRAME 这一虚拟内存上，因为在建立用户页表的时候，似乎就已经这么做过了
  // 所以此处做的修改，其实修改已经同步到 trapframe 上了
  p->trapframe->kernel_satp = r_satp();         // kernel page table
  p->trapframe->kernel_sp = p->kstack + PGSIZE; // process's kernel stack，这个值是建立的时候就准备好的，不过是在内核的地址空间之中
  p->trapframe->kernel_trap = (uint64)usertrap;
  p->trapframe->kernel_hartid = r_tp();         // hartid for cpuid()

  // set up the registers that trampoline.S's sret will use
  // to get to user space.
  
  // set S Previous Privilege mode to User.
  // 准备从 S 态切换到 U 态
  unsigned long x = r_sstatus();
  x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE; // enable interrupts in user mode
  w_sstatus(x);

  // set S Exception Program Counter to the saved user pc.
  // 就这边来看， epc 还是空值
  w_sepc(p->trapframe->epc);

  // tell trampoline.S the user page table to switch to.
  // 从 p->pagetable 中攫取所对应的页表起始地址
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which 
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  // 梳理：自 forkret 函数跳转到 trampoline 之中，同时告诉 trampoline 所对应的代码自己的页表根地址在哪里，以及 trapframe 位置

  // 虚拟地址计算偏移量，在 trampoline.S 中可以看到结果
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
}
```
下一步，跳转到`trampoline`位置的函数 userret 位置：不过这一次似乎是 forkret 进入后触发的，其实和我们在写 NestedSGX 初次进入到用户态一样，复用 syscall exit 部分的代码似乎是一个不成文的小技巧了。
```C
.globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        # 将 satp 的值替换成 user page table，这是第二个参数
        csrw satp, a1
        # 刷一下 TLB ，让更新能够被整个系统都知道
        sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        # 从 p->trampframe 中的位置，利用相对偏移来找到先前记录的 a0 寄存器的值
        # a0: trampframe
        # a0 in trampframe
        ld t0, 112(a0)
        # a0 可能之后要被用到，姑且先把它存放到 sscratch 寄存器里吧
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        # 不过第一次进来的时候这些值似乎都没有怎么被初始化过
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
    # TRAPFRAME 本来是 a0 ，未来的 sscratch 的值应该就是 TRAPFRAME 了
        csrrw a0, sscratch, a0
        
        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        # 跳转到 sepc 所展示的地址位置
        # 还没进去的时候，指向的位置是 p->trapframe->epc 地址
        # 而巧合的是， initcode 正好是从 0 虚拟地址开始进行拷贝的
        # 由于第一次进入时的 trampframe 并没有对相关寄存器做初始化，所以这边的寄存器值几乎都是乱码，即 0x50505050 状态我们
        # 的 initcode 函数可能也会因此特地写的很简单
        sret
```
之后，控制流将会进入到`initcode`程序中，因为 sepc 正好指向 0 了。
```shell
user/initcode.o:     file format elf64-littleriscv


Disassembly of section .text:

0000000000000000 <start>:
#include "syscall.h"

# exec(init, argv)
.globl start
start:
        la a0, init
   0:	00000517          	auipc	a0,0x0
   4:	00050513          	mv	a0,a0
        la a1, argv
   8:	00000597          	auipc	a1,0x0
   c:	00058593          	mv	a1,a1
        li a7, SYS_exec
  10:	00700893          	li	a7,7
        ecall
  # 准备调用 SYS_exec 了，给 a7 寄存器设置了 SYS_exec 这个值，不过暂时不知道这边的参数有什么用意？
  14:	00000073          	ecall

0000000000000018 <exit>:

# for(;;) exit();
exit:
        li a7, SYS_exit
  18:	00200893          	li	a7,2
        ecall
  1c:	00000073          	ecall
        jal exit
  20:	ff9ff0ef          	jal	18 <exit>

0000000000000024 <init>:
  24:	696e692f          	.word	0x696e692f
  28:	Address 0x28 is out of bounds.


000000000000002b <argv>:
	...
```
此处的 ecall 调了以后，控制流自然而然地就走向了`trampoline`之中了，之后进去就从函数 uservec 进去。这一点还是比较清楚的。
```shell
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
        # sscratch 是在第一次就通过 csrrw 写入 trapframe 值的
        csrrw a0, sscratch, a0

        # save the user registers in TRAPFRAME
        # 把这些寄存器上下文全部存放到 TRAPFRAME 之中
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

	# save the user a0 in p->trapframe->a0
        # 再把 a0 传输进来的值一并写到 trapframe 上，以后再拿来用
        # 目前 a0 还是指向 trapframe 吧
        csrr t0, sscratch
        sd t0, 112(a0)

        # restore kernel stack pointer from p->trapframe->kernel_sp
        ld sp, 8(a0)

        # make tp hold the current hartid, from p->trapframe->kernel_hartid
        ld tp, 32(a0)

        # load the address of usertrap(), p->trapframe->kernel_trap
        # 跳转到 usertrap 位置继续运行
        ld t0, 16(a0)

        # restore kernel page table from p->trapframe->kernel_satp
        # 切换了一下页表进入到内核地址空间，切换之后， a0 似乎就没有用了，因为
        ld t1, 0(a0)
        csrw satp, t1
        sfence.vma zero, zero

        # a0 is no longer valid, since the kernel page
        # table does not specially map p->tf.

        # jump to usertrap(), which does not return
        jr t0
```
我们来查看 usertrap 的情况，如下所示：
```C
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;
  // 检查一下 trap 的类型是否还是 usermode 触发的
  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  // 加上这个不分，让 kernel 触发的 trap 也能被捕获到
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  // sret 将会重新跳转到 sepc 位置， sepc 是系统调用发生时自动保留的位置
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
    intr_on();
    // 进入到 syscall 表格之中
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
我们继续看向 syscall 函数，在这个函数之后，将会通过 usertrapret 返回原处：特别的，系统调用所需的参数其实都存放到了 trapframe 之中，可以相对方便地进行取用
```C
void
syscall(void)
{
  int num;
  struct proc *p = myproc();
  // 确认 num ，在之前我们写了 SYS_exec ，对应的是 a7 寄存器
  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // 调用 syscalls[num]() 函数，返回值写入到 trapframe 的 a0 寄存器之中
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
特别的， syscall 的初始化写法很有意思：syscalls 是一个数组，每个位置都写着某个号所对应的函数地址
```C
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
};
```
在我们 initcode 之中，会调用函数 sys_exec ，我们继续跟踪：
```C
uint64
sys_exec(void)
{
  char path[MAXPATH], *argv[MAXARG];
  int i;
  uint64 uargv, uarg;
  // 传输给 sys_exec 的 a0 和 a1 其实应该都是地址
  // 最终的目的应该是利用这两个地址来通过 pagetable 遍历，把值获得到
  // 不能直接以内核身份通过加载 pagetable 来实现对数据的攫取，但是可以另有方式，通过遍历页表来实现
  if(argstr(0, path, MAXPATH) < 0 || argaddr(1, &uargv) < 0){
    return -1;
  }
  memset(argv, 0, sizeof(argv));
  for(i=0;; i++){
    if(i >= NELEM(argv)){
      goto bad;
    }
    if(fetchaddr(uargv+sizeof(uint64)*i, (uint64*)&uarg) < 0){
      goto bad;
    }
    if(uarg == 0){
      argv[i] = 0;
      break;
    }
    argv[i] = kalloc();
    if(argv[i] == 0)
      goto bad;
    if(fetchstr(uarg, argv[i], PGSIZE) < 0)
      goto bad;
  }
  // 在上面已经完全通过 a0 和 a1 寄存器获取了
  // 这玩意我们分析过了，不管他了
  int ret = exec(path, argv);

  for(i = 0; i < NELEM(argv) && argv[i] != 0; i++)
    kfree(argv[i]);

  return ret;

 bad:
  for(i = 0; i < NELEM(argv) && argv[i] != 0; i++)
    kfree(argv[i]);
  return -1;
}
```
首先我们来查看参数获取的流程： n 代表选取哪一个寄存器进行传参，使用起来还是比较方便的：
```C
// Fetch the nth word-sized system call argument as a null-terminated string.
// Copies into buf, at most max.
// Returns string length if OK (including nul), -1 if error.
int
argstr(int n, char *buf, int max)
{
  uint64 addr;
  // 说白了就是获取， p->trapframe->a0 中存放的参数所对应的地址
  if(argaddr(n, &addr) < 0)
    return -1;
  // 利用这个地址，来调用 fetchstr 函数
  return fetchstr(addr, buf, max);
}

// Retrieve an argument as a pointer.
// Doesn't check for legality, since
// copyin/copyout will do that.
int
argaddr(int n, uint64 *ip)
{
  *ip = argraw(n);
  return 0;
}

static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}


// Fetch the nul-terminated string at addr from the current process.
// Returns length of string, not including nul, or -1 for error.
int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  // 通过 copyinstr 遍历 p->pagetable ，来读取信息到 buf 之中
  int err = copyinstr(p->pagetable, buf, addr, max);
  if(err < 0)
    return err;
  return strlen(buf);
}
```
这边做好之后，回到 usertrapret：和我们第一次 forkret 初始化的步骤几乎一样

3. 一些有意思的现象：
- 需要在读过符号表之后，才能看到 C 语言层面的描述，否则就只有汇编
- 南大教给我的调试方法非常有用

到这边全部的调试流程都走了一遍，感觉还是很清楚的哈哈哈，代码和 book 都看完了，可以考虑做下面的实验了。

## 实验记录
### riscv assembly
1. Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?
- a2 holds 13 in main's call to printf.

2. Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)
```shell

user/_call:     file format elf64-littleriscv


Disassembly of section .text:

0000000000000000 <g>:
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd	s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld	s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,-16
  10:	e422                	sd	s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld	s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret

000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7a850513          	addi	a0,a0,1960 # 7d0 <malloc+0x102>
  30:	00000097          	auipc	ra,0x0 # ra is set to 34.
  34:	5e6080e7          	jalr	1510(ra) # 616 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	274080e7          	jalr	628(ra) # 2ae <exit>
```