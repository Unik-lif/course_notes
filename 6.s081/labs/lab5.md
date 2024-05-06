## 实验概览：
这个实验似乎在前头的课程中，franks就已经简单地提过一嘴。

### Eliminate allocation from sbrk
本质上是对 sbrk 的行为做修改，将其原本作为静态分配空间的方式修改成出现 page fault 之后再进行处理的模式。

会破坏的地方在于，本来因为 sbrk 操作能够做分配的部分出现了访问，从而造成了 pagefault ，因为我们并没有真正地对其进行内存地址空间的分配。

之后得到的效果如下所示：
```
init: starting sh
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x0000000000001272 stval=0x0000000000004008
panic: uvmunmap: not mapped

(gdb) bt
#0  panic (s=s@entry=0x800080f8 "uvmunmap: not mapped") at kernel/printf.c:125
#1  0x0000000080001386 in uvmunmap (pagetable=0x87f75000, va=va@entry=0, npages=npages@entry=20, do_free=do_free@entry=1) at kernel/vm.c:186
#2  0x0000000080001628 in uvmfree (pagetable=pagetable@entry=0x87f75000, sz=sz@entry=81920) at kernel/vm.c:298
#3  0x0000000080001bf8 in proc_freepagetable (pagetable=0x87f75000, sz=81920) at kernel/proc.c:195
#4  0x0000000080001c2e in freeproc (p=p@entry=0x80012038 <proc+720>) at kernel/proc.c:143
#5  0x000000008000234c in wait (addr=0) at kernel/proc.c:429
#6  0x0000000080002c8a in sys_wait () at kernel/sysproc.c:38
#7  0x0000000080002bcc in syscall () at kernel/syscall.c:140
#8  0x00000000800028b6 in usertrap () at kernel/trap.c:67
#9  0x0000000000000ac8 in ?? ()
```
产生原因在 backtrace 下也能做一个大致的猜测。似乎并不是源自访问了没有分配的地址，而是因为按图索骥在 freeproc 的时候对内存出现了访问导致出错。

从 log 打印来看似乎印证了我们的猜想，还是很明显的：
```C
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  printf("addr: %p\n", addr);
  /*
  if(growproc(n) < 0)
    return -1;
  */
  myproc()->sz = myproc()->sz + n;
  printf("sz: %p\n", myproc()->sz);
  return addr;
}

// the log info of echo hi.
/*
init: starting sh
$ echo hi
addr: 0x0000000000004000
sz: 0x0000000000014000
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x0000000000001272 stval=0x0000000000004008
panic: uvmunmap: not mapped
*/
```
### Lazy allocation
首先 pagefault 存在两种成因，一种是 load 一种是 store ，分别对应 13 或者 15 错误号。我们就此可以做一些简单的处理。

特别的，在 stvec 寄存器的 DIRECT 模式下 riscv 中的中断和异常的处理入口似乎都是在 stvec 寄存器上。因此我们直接在 usertrap 这边改就好了。

照着 uvmalloc 的代码来写，记得做好 mapping ，以及在 uvmfree 的时候跳过没有分配过的页，就能很轻松地完成这个任务。

**注意使用 PGROUNDDOWN ，因为 PGROUNDUP 所指向的是下一个页，当前页的话应该使用 PGROUNDDOWN**

可能是有一段时间没写了，我对于 PGROUNDDOWN 和 PGROUNDUP 没有以前那么敏感了，确实不应该没有意识到这个错误。
### Lazytests and Usertests
#### 对于 lazytests：
这个不难，主要要多多使用 vmprint 来找可能出错的位置。

对于 sbrk(n) 中的 n 小于 0 的情况，我们可以采用 uvmunmap 来释放掉多余的内存。

然后多做一些 continue ，跳过并没有实际分配的页地址空间就好了。

#### 对于 usertests：
有几个不是那么好过：

首先是一开始的 countfree ， kalloc 有可能会耗尽，为此你需要在耗尽的时候 killed 掉进程。

第二个似乎是 sbrkbugs 了，看起来没这么好做，大体上是测试中把 usertests 全部 free 掉了，包括指令相关的地址，因此程序似乎就没法往下跑了。在 pagefault 这边，可能需要特殊做一些处理。

看起来最后在这个位置上死循环了，说白了就是把内存中的指令也给清空了，于是 sepc 和 stval 了值都是一样的。
```
usertrap(): unexpected scause 0x000000000000000c pid=6      
            sepc=0x0000000000005622 stval=0x0000000000005622
```
对应的异常是 Instruction page fault ，为了能够正确处理，需要给 page fault 加上一个 case （除了 load 与 store 之外），然后通过 sz 的判别成功退出。现在除了上面提到的 13 和 15 两个错误号，我们还要添加 12。

第三个似乎是 sbrkarg ，在实验文档中似乎有针对这个的描述：

- Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.

问题出现在 write 和 pipe 等系统调用必须要对对应的 addr 进行访问，可是他们又不会经过 pagefault 的方式来进行触发，为此我们需要在 write 和 pipe 等部分系统调用中加上原本用于处理 pagefault 的代码。
```C
  uint64 mem = 0;
  for(int i = 0; i < n; i += PGSIZE){
    if(walkaddr(myproc()->pagetable, addr + i) == 0){
      mem = kalloc();
      if(mem == 0){
        myproc()->killed = 1;
        exit(-1);
      }
      memset(mem, 0, PGSIZE);
      mappages(myproc()->pagetable, PGROUNDDOWN(addr + i), PGSIZE, mem, PTE_W|PTE_X|PTE_R|PTE_U);
    }
  }
```
但是只是这么做是不够的，不同的情况在于，addr + i 这个地址需要低于 myproc()->sz ，这样就能满足了。
```C
  uint64 mem = 0;
  for(int i = 0; i < n; i += PGSIZE){
    if(walkaddr(myproc()->pagetable, addr + i) == 0 && (addr + i) < myproc()->sz){
      mem = kalloc();
      if(mem == 0){
        myproc()->killed = 1;
        exit(-1);
      }
      memset(mem, 0, PGSIZE);
      mappages(myproc()->pagetable, PGROUNDDOWN(addr), PGSIZE, mem, PTE_W|PTE_X|PTE_R|PTE_U);
    }
  }
```
第四个是 stacktest 这个，面对的问题同样在这边做了解释：
 
- Handle faults on the invalid page below the user stack.

加上这个检查就好了：
```C
|| vm_pg < PGROUNDUP(myproc()->trapframe->sp)
```

OK 做到这边就搞定了。

```
== Test running lazytests == 
$ make qemu-gdb
(4.8s) 
== Test   lazy: map == 
  lazy: map: OK 
== Test   lazy: unmap == 
  lazy: unmap: OK 
== Test usertests == 
$ make qemu-gdb
(95.4s) 
== Test   usertests: pgbug == 
  usertests: pgbug: OK 
== Test   usertests: sbrkbugs == 
  usertests: sbrkbugs: OK 
== Test   usertests: argptest == 
  usertests: argptest: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: sbrkfail == 
  usertests: sbrkfail: OK 
== Test   usertests: sbrkarg == 
  usertests: sbrkarg: OK 
== Test   usertests: stacktest == 
  usertests: stacktest: OK 
== Test   usertests: execout == 
  usertests: execout: OK 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: rwsbrk == 
  usertests: rwsbrk: OK 
== Test   usertests: truncate1 == 
  usertests: truncate1: OK 
== Test   usertests: truncate2 == 
  usertests: truncate2: OK 
== Test   usertests: truncate3 == 
  usertests: truncate3: OK 
== Test   usertests: reparent2 == 
  usertests: reparent2: OK 
== Test   usertests: badarg == 
  usertests: badarg: OK 
== Test   usertests: reparent == 
  usertests: reparent: OK 
== Test   usertests: twochildren == 
  usertests: twochildren: OK 
== Test   usertests: forkfork == 
  usertests: forkfork: OK 
== Test   usertests: forkforkfork == 
  usertests: forkforkfork: OK 
== Test   usertests: createdelete == 
  usertests: createdelete: OK 
== Test   usertests: linkunlink == 
  usertests: linkunlink: OK 
== Test   usertests: linktest == 
  usertests: linktest: OK 
== Test   usertests: unlinkread == 
  usertests: unlinkread: OK 
== Test   usertests: concreate == 
  usertests: concreate: OK 
== Test   usertests: subdir == 
  usertests: subdir: OK 
== Test   usertests: fourfiles == 
  usertests: fourfiles: OK 
== Test   usertests: sharedfd == 
  usertests: sharedfd: OK 
== Test   usertests: exectest == 
  usertests: exectest: OK 
== Test   usertests: bigargtest == 
  usertests: bigargtest: OK 
== Test   usertests: bigwrite == 
  usertests: bigwrite: OK 
== Test   usertests: bsstest == 
  usertests: bsstest: OK 
== Test   usertests: sbrkbasic == 
  usertests: sbrkbasic: OK 
== Test   usertests: kernmem == 
  usertests: kernmem: OK 
== Test   usertests: validatetest == 
  usertests: validatetest: OK 
== Test   usertests: opentest == 
  usertests: opentest: OK 
== Test   usertests: writetest == 
  usertests: writetest: OK 
== Test   usertests: writebig == 
  usertests: writebig: OK 
== Test   usertests: createtest == 
  usertests: createtest: OK 
== Test   usertests: openiput == 
  usertests: openiput: OK 
== Test   usertests: exitiput == 
  usertests: exitiput: OK 
== Test   usertests: iput == 
  usertests: iput: OK 
== Test   usertests: mem == 
  usertests: mem: OK 
== Test   usertests: pipe1 == 
  usertests: pipe1: OK 
== Test   usertests: preempt == 
  usertests: preempt: OK 
== Test   usertests: exitwait == 
  usertests: exitwait: OK 
== Test   usertests: rmdot == 
  usertests: rmdot: OK 
== Test   usertests: fourteen == 
  usertests: fourteen: OK 
== Test   usertests: bigfile == 
  usertests: bigfile: OK 
== Test   usertests: dirfile == 
  usertests: dirfile: OK 
== Test   usertests: iref == 
  usertests: iref: OK 
== Test   usertests: forktest == 
  usertests: forktest: OK 
== Test time == 
time: OK 
Score: 119/119
```