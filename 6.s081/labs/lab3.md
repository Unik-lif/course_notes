## 实验概览
目的是搞清楚`pgtable`的具体工作流程，我反正遵循一个原则，代码必须看懂了再开始尝试做。最好情况下是一行都不要落下，特别细节的搞清楚了也无所谓，很多架构就原则上、思想上是类似的，只不过其`ISA`与相关绑定的硬件生态会有所不一。

## 源码解读：
还是从最开始的`main.c`代码结构开始：我们很轻松的就能注意到，针对内核环境的地址空间初始化同样还是`hartid`为`0`的`CPU`去做这件事情，其他进程并不需要像`BSP`那样来初始化内核地址空间，也不需要初始化页表，页表的话是在`BSP`中搞好，然后别人的核把页表机制开起来就可以了。

这件事是否正确仍然需要做一下之后验证，我们暂时不管。一开始我们的分析可以立足`CPUS=1`的情况，之后我们再尝试扩展到多核。
```C
    // BSP cores.
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging

    // In different branch of hartid => AP cores.
    // AP cores.
    kvminithart();    // turn on paging
}
```
### kinit函数
不得不说`xv6`实现的非常简单清楚：先初始化锁，然后开辟空闲地址空间。

我们看到下面的这个函数，需要注意到`end`是`xv6`代码段的终结，`PHYSTOP`是内存的终结。其中的`initlock`函数将会对`kmem.lock`这个锁做简单的声明与初始化：

我们可以看到这边是做了`kmem`的初始化，表示内核空间确实已经被初始化，当然这件事是由`bsp`来去做，此外利用了`freerange`函数将内核结束地址空间之后的内存全部纳入`free`地址空间考虑范畴内。
```C
struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  // the end here is where the kernel ends, the rest of the memory
  // should be put into use.
  freerange(end, (void*)PHYSTOP);
}

void
initlock(struct spinlock *lk, char *name)
{
  lk->name = name;
  lk->locked = 0;
  lk->cpu = 0;
}

void
freerange(void *pa_start, void *pa_end)
{
  // virtual memory has not been established yet, so doing this won't cause a problem.
  char *p;
  // roundup the end, to get the next page.
  p = (char*)PGROUNDUP((uint64)pa_start);
  // use kfree(P) here, the kfree is still a very basic version, simply set the page where p points to as 1
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}
```
对于这边的`kfree`，其代码如下所示：这是一个链表增加的过程，挺好解释的。
```C
// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  // check whether the pa here is valid.
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  // lock the kmem.
  // kmem is orchestrated as a freelist, so before modifying the memory, we should acquire a lock first.
  // update the freelist, since we've freed a page, the freelist now is appended with a new page node.
  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```
### kvminit函数
对于`kvminit`函数，其结构如下：其名字是对内核虚拟地址空间做虚拟化，我们逐步分析。

`kalloc`看起来是很简单的`kfree`的反向操作，用这个分配了页表空间。`kvmmap`是映射的关键函数，在这边逐一对部分重要`IO`设备，以及内核源码与数据做了设置，我们需要仔细研究。
```C
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

/*
 * create a direct-map page table for the kernel.
 */
void
kvminit()
{
  kernel_pagetable = (pagetable_t) kalloc();
  // set it to zero.
  memset(kernel_pagetable, 0, PGSIZE);

  // uart registers
  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
```
`kvmmap`函数信息如下所示：它使用了`mappages`这个函数作为主体，而`mappages`则以`pagewalk`为主体，代码注释中我们另做解析：
```c
// add a mapping to the kernel page table.
// only used when booting.
// does not flush TLB or enable paging.
void
kvmmap(uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(kernel_pagetable, va, sz, pa, perm) != 0)
    panic("kvmmap");
}

// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa. va and size might not
// be page-aligned. Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  // in fact it is a little strange to choose PGROUNDDOWN here.
  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  // pgtable walk to fetch the specific page table
  for(;;){
    // walk the pagetable, try to find the corresponding pte of a.
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    // before the mapping of the pages
    // the pte should be seen as invalid, so the PTE_V should not be set.
    if(*pte & PTE_V)
      panic("remap");
    // page addree to PTE (get the higher position bit value), using "|" together to form a pte.
    // perm is the permission the kernel has set.
    *pte = PA2PTE(pa) | perm | PTE_V;
    // allocate until we reach the last, we should break.
    // look back to the PGROUNDDOWN we set to va + size - 1
    // therefore for size smaller than a page, we still can allocate one page.
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}

// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    // pagetable => some level of pagetable.
    // always change pagetable, this one got 512 entries, but to reduce the memory we should allocate
    // use the dynamic way to allocate the memory.

    // get the offset value from va.
    pte_t *pte = &pagetable[PX(level, va)];
    // have the entry, and the page table entry is also valid.
    if(*pte & PTE_V) {
      // get the physical address of next level of pagetable.
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      // not exist. or not valid.
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  // get the ultimate pte address, level is equal to 0 now.
  return &pagetable[PX(0, va)];
}
```
对于`pagetable_t`来说，其数据结构如下所示：其实它真的只是一个数组。其他的一些定义我们可以参考`riscv.h`中的设置，也可以参考手册。
```C
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access

// shift a physical address to the right place for a PTE.
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)

#define PTE2PA(pte) (((pte) >> 10) << 12)

// the lower 10 bits are FLAGS.
#define PTE_FLAGS(pte) ((pte) & 0x3FF)

// extract the three 9-bit page table indices from a virtual address.
#define PXMASK          0x1FF // 9 bits => to get the whole info of a level.
// PGSHIFT is 12, adding 9 * level to get the right shift offset of the corresponding level.
#define PXSHIFT(level)  (PGSHIFT+(9*(level)))
#define PX(level, va) ((((uint64) (va)) >> PXSHIFT(level)) & PXMASK)

// one beyond the highest possible virtual address.
// MAXVA is actually one bit less than the max allowed by
// Sv39, to avoid having to sign-extend virtual addresses
// that have the high bit set.
#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))

typedef uint64 pte_t;
typedef uint64 *pagetable_t; // 512 PTEs
```
好像，也没什么好说的，看一看`specification`就好了。

### kvminithart函数、
```C
// Switch h/w page table register to the kernel's page table,
// and enable paging.
void
kvminithart()
{
  w_satp(MAKE_SATP(kernel_pagetable));
  sfence_vma();
}
```
也没干什么，我们刚刚分配了页表，起始的地址被记作`kernel_pagetable`，然后把它设置到`satp`寄存器上，仅此而已。看看手册就行，记得`MODE`高位做好设置。之后，把`TLB`表清空了就行。

到这边，这代码分析是搞定了，我们可以推接下来的实验了。
## 其他部分（实验后发现需要继续阅读）
内核部分就内存空间的初始化我们是看到了，现在我们需要看一下用户空间这件事情是怎么做到的。

实验二对于用户进程所对应的地址空间做了一些相应的研究，我们也需要对这一部分代码做一个简单的解读。

尝试分析`procinit`函数，如下所示：
```C
// initialize the proc table at boot time.
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  // go through the proc array
  for(p = proc; p < &proc[NPROC]; p++) {
      // initialize every proc's lock.
      initlock(&p->lock, "proc");

      // Allocate a page for the process's kernel stack.
      // Map it high in memory, followed by an invalid
      // guard page.

      // allocate the kernel stack of the process.
      char *pa = kalloc();
      if(pa == 0)
        panic("kalloc");
      // I have a test on p - proc value, it is simply a array ranged from 0..10
      // act as the kernel stack of these processes.
      uint64 va = KSTACK((int) (p - proc));
      kvmmap(va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
      p->kstack = va;
  }
  kvminithart();
}
```
挺直白的，在注释上写了。

另外还有一个部分，反映程序跑起来的情况：
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
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      // fetch current proc.
      acquire(&p->lock);
      // ensure the state of the process is runnable.
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        // switch current cpu's context to the process's context
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
此处的`mycpu`对`cpu`数组做了一次简单的读取工作：找到需要处理的对应`cpu`数据结构。
```C
// Must be called with interrupts disabled,
// to prevent race with process being moved
// to a different CPU.
int
cpuid()
{
  int id = r_tp();
  return id;
}

// Return this CPU's cpu struct.
// Interrupts must be disabled.
struct cpu*
mycpu(void) {
  int id = cpuid();
  struct cpu *c = &cpus[id];
  return c;
}
```
切换工作的时候，调用了`swtch`函数，我们陷入进去看一下：
```C
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
其实也很简单，把当前状态存储到当前`cpu`所对应的`context`之中，并读取`process`所对应的`context`信息。
## 实验
### 实验一：page table print
可以直接参考`freepagewalk`时的做法，很轻松就能实现。注意这边的递归可能处理起来比较麻烦，因此我使用了一个辅助函数，并且用了一个`level`变量来帮忙打印层级。

### 实验二：为每个进程分配一个kernel page table
这个实验似乎是为了性能和可用性来考虑，为了让`kernel`能够轻松地去获得用户进程的变量地址，需要为每个用户进程做一个内核地址空间，使得内核可以直接对相关变量进行访问，而不需要转换成物理地址再去处理。

在这一步我们仅仅需要为每个用户进程生成一个内核页表的副本，在下一步我们才会完成地址翻译的加速工作。

在做这个实验的时候，需要判断在`freeproc`的时候，是否完全把页表给释放干净了，这一点可以通过前面实现的`vmprint`来完成，不过似乎在很多情况下，情况并不如我们所愿。

打印出来的结果是这个样子的：
```
page table 0x0000000087e87000                                                                                                                                                                              
..0: pte 0x0000000021fa1801 pa 0x0000000087e86000                                                                                                                                                          
.. ..16: pte 0x0000000021fa0c01 pa 0x0000000087e83000                                                                                                                                                      
.. ..96: pte 0x0000000021fa0801 pa 0x0000000087e82000                                                                                                                                                      
.. ..97: pte 0x0000000021f9bc01 pa 0x0000000087e6f000                                                                                                                                                      
.. ..128: pte 0x0000000021fa1401 pa 0x0000000087e85000                                                                                                                                                     
..2: pte 0x0000000021f9c001 pa 0x0000000087e70000                                                                                                                                                          
.. ..0: pte 0x0000000021f9c401 pa 0x0000000087e71000                                                                                                                                                       
.. ..1: pte 0x0000000021f9c801 pa 0x0000000087e72000                                                                                                                                                       
.. ..2: pte 0x0000000021f9cc01 pa 0x0000000087e73000                                                                                                                                                       
.. ..3: pte 0x0000000021f9d001 pa 0x0000000087e74000                                                                                                                                                       
.. ..4: pte 0x0000000021f9d401 pa 0x0000000087e75000                                                                                                                                                       
.. ..5: pte 0x0000000021f9d801 pa 0x0000000087e76000                                                                                                                                                       
.. ..6: pte 0x0000000021f9dc01 pa 0x0000000087e77000                                                                                                                                                       
.. ..7: pte 0x0000000021f9e001 pa 0x0000000087e78000                                                                                                                                                       
.. ..8: pte 0x0000000021f9e401 pa 0x0000000087e79000                                                                                                                                                       
.. ..9: pte 0x0000000021f9e801 pa 0x0000000087e7a000
```
其实一开始还真没有意识到，其实这边没有任何`leaf page`了，我一直以为我有哪些东西没有清空掉，找了很久但还是没有找到，现在也就是说，我们的`free`已经把所有的`PTE`项给清空了。那么其实只要做一件事情就好，把这边的`pde`给清理了。

那么我们直接去找有没有相关的函数，结果发现函数freewalk正好能用，需要注意的是，它自己会把自己给清空干净，所以不需要再清空了，清空后也不能打印了，否则可能会得到这个值：
```
after free walk
page table 0x0000000087e87000
..1: pte 0x0101010101010101 pa 0x0404040404040000
scause 0x000000000000000d
sepc=0x000000008000198a stval=0x0404040404040000
panic: kerneltrap
```
看到`pte`和`pa`中的数值，你可能马上回想起`kfree`中的设置。总而言之，它已经被清空了。

### 实验三：
这个实验似乎是要把`copyin`函数变得很快，这样就不需要通过跨`pagetable`的地址翻译，

这里使用的策略似乎是，需要依赖用户的虚拟地址和内核虚拟地址不相互重叠，对于用户的虚拟地址来说，`xv6`是从`0`开始计量的。

我们先看看原本为什么`copyinstr`之类的能够成功。
```C
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    // pa0
    va0 = PGROUNDDOWN(dstva);
    // get the corresponding physical address.
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    // va0 is the start address of the virtual page.
    // dstva is the position we want to copy in user page table.
    // here we will copy max for PGSIZE - (dstva - va0) length of bytes.
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    // copy the physical address to this physical address.
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}

// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```
根据上面的注释，我们可以看到总体来说要做的事是逐个对地址进行翻译，翻译成物理地址，之后再进行拷贝的操作，不过这个过程看起来相当复杂，因为每复制一个页面，都需要在页表空间中作一个设置。

为了降低这边工作的复杂性，需要在`kernel_pagetable`中添加用于用户自身虚拟地址翻译的一部分，这样一来，内核就不用通过`pgtbl_walk`来对具体的字节位置进行翻译。

我们就可以比较顺畅地使用下面的两个函数：
```C
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  struct proc *p = myproc();

  // outfit the process virtual address space size.
  // or overflow.
  if (srcva >= p->sz || srcva+len >= p->sz || srcva+len < srcva)
    return -1;
  // copy srcva to dst, with len bytes size.
  memmove((void *) dst, (void *)srcva, len);
  stats.ncopyin++;   // XXX lock
  return 0;
}

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  struct proc *p = myproc();
  char *s = (char *) srcva;
  
  stats.ncopyinstr++;   // XXX lock
  for(int i = 0; i < max && srcva + i < p->sz; i++){
    dst[i] = s[i];
    if(s[i] == '\0')
      return 0;
  }
  return -1;
}
```
首先我们看向用户虚拟地址空间是什么时候被设置的，看起来一开始初始化的时候是空的一个页表，似乎不太构成什么参考价值，真正开始起作用似乎是从`userinit`这边开始，我们的目标是在进程所对应的内核地址中，先亦步亦趋地将映射到进程的`kernel_pagetable`之中去。

这件事情看上去不是特别难，我们暂时可以先简单考虑一下先前所说的`PLIC`的范围限制，这个限制的大意似乎是：
- 对于小于`PLIC`范围的虚拟地址空间，我们可以对其进行加速。因此，在上面对于kernel_pagetable的映射之中，我们需要判断涉及的虚拟地址数值范围，如果大于`PLIC`了，就不要再做`mappings`了，否则容易造成混淆，也就是说一个虚拟地址可能会映射到两个物理地址上。我们知道，页表的映射允许一个物理地址映射到多个虚拟地址，但是反过来是不行的。
- 对于大于`PLIC`范围的虚拟地址空间，我们还是按照原本的方案来做。



长期没写，寄了，我们可能得重新开始干。

我们首先需要搞清楚，进行内存分配的函数是什么样子的：
```C
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa. va and size might not
// be page-aligned. Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    // walk 函数如果返回值为0，则 return -1
    // 特别的，这边还是有 alloc 的情况下，说明空间满了
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    // 如果已经分配过了， PTE_V 会被设置
    // 这边居然直接 panic 了，似乎是不应该的，在某些场景下我们需要对页表权限做一些修改
    if(*pte & PTE_V)
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}

// walk函数如下所示
// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");
  // 建议画一张图，这边的代码是对的
  // 有 pagetable 的结构，至少说明 level 2 所对应的 PTE 这边是存在页的
  // 从 level 2 开始向下遍历
  for(int level = 2; level > 0; level--) {
    // 对 va 中所对应 level 2 的部分虚拟地址做解析，得到对应是页表的第几项
    pte_t *pte = &pagetable[PX(level, va)];
    // 选取下一级的 base 地址，之后继续利用 va 挑选对应的第几项
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      // 如果 alloc 没有被设置为 0，或者 kalloc 中没有富余的空间
      // 我们就直接返回一个 0 
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      // 如果有空间，则把刚分配的页进行设置
      memset(pagetable, 0, PGSIZE);
      // 对该页所对应的 pte 进行写操作，便于以后遍历使用
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  // 返回了 level 0 所对应的 PTE 的地址
  return &pagetable[PX(0, va)];
}

// Look up a virtual address, return the physical address,
// or 0 if not mapped.
// Can only be used to look up user pages.
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  // 特别注意这边，对于不包含 PTE_U 的 pte 项，自动返回 0
  // 因此如果我们的 pte 是对内核打开的，我们不应该使用这个函数对地址进行遍历
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}

// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");
  // 从 va 开始来逐个页面地释放掉
  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    // 可以直接找到 pte 所对应的物理地址
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    // 内核地址空间，对 pte 所对应的物理地址的地方
    // 进行写 0 的操作
    // 如果有其他页表一并访问了 pte ，在之后观测也将是 0
    *pte = 0;
  }
}
```
注意到一个现象，先前在`kernel_pagetable`中插入的用户所对应的页表还是需要人工手动去`free`掉的，否则会导致最后的`freewalk`没法较为彻底地清空掉映射的内存页。

这听起来似乎是应该的？我们重新在`uvmfree`中加上去除掉映射的函数操作。

一些其他遇到的问题：
1. `CLINT`这个应该可以不做`maps`
2. 在写`exec`的进程所对应的`kernel_pagetable`要稍微小心一点，`uvminit`函数只会被调用一遍，因此写代码要小心。
3. 有一个问题，目前不知道怎么解决：
```
[exec] p->pid: 11
[proc_pagetable]:
[proc_kpgtblinit]:
proc_kvmmap: va: 0x0000000010000000 pa: 0x0000000010000000
test: -1
panic: proc_kvmmap
```
在这边出现了`panic`的情况，似乎是`kalloc`没有富余空间了？结果看到似乎本身是`k_pgtable`这个地方没有被合适地分配到。

不管怎么样，似乎说明所用的内存空间太多了，得查看`kalloc`和`kfree`的一些基本逻辑，或者说，研究一下是否有必要完全新开一个`old_kernelpagetable`。
```
[proc_kpgtblinit]:              
[proc_pagetable]:                       
p->pid: 1                              
[uvminit]:                        
[exec] p->pid:                  
[proc_pagetable]              
[proc_kpgtblinit]:                        
[proc_freepagetable]:                   
[uvmfree]:                  
[uvmfree]: below PLIC                     
[proc_freekpgtbl]:
init: starting sh
fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
[exec] p->pid: 2
[proc_pagetable]:
[proc_kpgtblinit]:
[proc_freepagetable]:
[uvmfree]:
[uvmfree]: below PLIC
[proc_freekpgtbl]:
$ usertests                             
fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
[exec] p->pid: 3
[proc_pagetable]:
[proc_kpgtblinit]:
[proc_freepagetable]:
[uvmfree]:
[uvmfree]: below PLIC
[proc_freekpgtbl]:
usertests starting
fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
[uvmdealloc]:
[proc_freepagetable]:
[uvmfree]:
[uvmfree]: below PLIC
[proc_freekpgtbl]:
test execout: fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
[uvmdealloc]:
[proc_freepagetable]:
[uvmfree]:
[uvmfree]: below PLIC
[proc_freekpgtbl]:
fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
[uvmdealloc]:
[uvmdealloc]:
[proc_freepagetable]:
[uvmfree]:
[uvmfree]: below PLIC
[proc_freekpgtbl]:
fork
[proc_kpgtblinit]:
[proc_pagetable]:
[uvmcopy]:
[uvmdealloc]:
```
感觉可能需要研究一下`uvmdealloc`和`uvmcopy`，感觉有可能是问题的触发点。

我仔细看了一下，很好玩：第一个测试用例似乎就用尽了全部的内存空间，因此如果我们的程序没有好好写，比如又分配了一个内核页表，这样就会直接跑崩。所以我们需要想出一个不需要依赖新设置内核页表的方法。
```C
// test the exec() code that cleans up if it runs out
// of memory. it's really a test that such a condition
// doesn't cause a panic.
void
execout(char *s)
{
  for(int avail = 0; avail < 15; avail++){
    int pid = fork();
    if(pid < 0){
      printf("fork failed\n");
      exit(1);
    } else if(pid == 0){
      // allocate all of memory.
      while(1){
        uint64 a = (uint64) sbrk(4096);
        if(a == 0xffffffffffffffffLL)
          break;
        *(char*)(a + 4096 - 1) = 1;
      }

      // free a few pages, in order to let exec() make some
      // progress.
      for(int i = 0; i < avail; i++)
        sbrk(-4096);
      
      close(1);
      char *args[] = { "echo", "x", 0 };
      exec("echo", args);
      exit(0);
    } else {
      wait((int*)0);
    }
  }

  exit(0);
}
```
此外原本的用来做`kernel_pgtbl`的方式似乎有点麻烦，我们先尝试继续把原本那个东西做好试试看。应该还是有办法来实现的。

之后我们再考虑用一些相对简单一些的方法。

我后来发现，原本的思路是有问题的，集中体现在：
1. 基本做不到及时跟进，虽然尝试亦步亦趋来给`kern_pagetable`进行映射，但是在很多情况下没有办法做到这一点。比如因为内存使用率很大导致`exec`分配内存，映射到一半就进入了`bad`分支，`kern_pagetable`明明还有很多页并没有映射，那么当我们亦步亦趋来做的时候，我们如何确认到底有哪些页已经被映射了，哪些页没有被映射呢？这件事情真的超级复杂。
2. 设计起来特别麻烦，我尝试修改其中的`bug`，发现问题越改越多，或许这说明本身我们的设计就是有一些问题的。


`copyin`函数在什么时候用：被`wrapper`包起来，在`either_copyin`中进行使用，这个函数在`fetch_addr`之中会被调用。

我们以看起来一个比较好的方式来进行重新代码撰写

特别的，`sbrk`存在`shrink`的情况，需要考虑在此时释放掉多余的内存。
```C
      // free a few pages, in order to let exec() make some
      // progress.
      for(int i = 0; i < avail; i++)
        sbrk(-4096);
```
看起来测例全部都能通过，不过存在有时候会阻塞的部分可能性，正在思考是否是实现上的问题。
```
test forktest: OK
test bigdir: OK
ALL TESTS PASSED
```

发现有一些边角料上的问题，尝试`debug`。

调试了较长时间，重构了一次`kvmcopy`，似乎是终于完成了这个任务。对于内存的有限操作以及调试获得的经验确实收获不小。

具体的思路是，首先就应该把`copyin`，`copyin_str`全部做替换，然后再开始我们的工作。

需要撰写两个函数来进行工作：`kvmcopy`负责拷贝，`kvmfree`负责关闭掉多余的页映射
```C
int
kvmcopy(pagetable_t pagetable, pagetable_t k_pagetable, uint64 start_va, uint64 end_va)
{
  // printf("[kvmcopy]: %p %p\n", start_va, end_va);
  uint64 pa = 0;
  int perm = 0;
  pte_t* pte_u = 0;
  pte_t* pte_k = 0;

  for(int i = start_va; i < end_va; i += PGSIZE){
    pte_u = walk(pagetable, i, 0);
    pte_k = walk(k_pagetable, i, 1);
    pa = PTE2PA(*pte_u);
    perm = PTE_FLAGS(*pte_u);
    
    *pte_k = PA2PTE(pa) | (perm & (~PTE_U)) | PTE_V;
  }
  return 0;
}

void
kvmfree(pagetable_t k_pagetable, uint64 start_va, uint64 end_va)
{
  // printf("[kvmfree]:%p %p\n", start_va, end_va);
  if(walkaddr_kern(k_pagetable, start_va) == 0) return;
  // printf("kvmfree here.\n");
  if(end_va < PLIC)
    uvmunmap(k_pagetable, start_va, (end_va - start_va)/PGSIZE, 0);
  else
    uvmunmap(k_pagetable, start_va, (PLIC - start_va)/PGSIZE, 0);
}
```
需要修改的地方确实是在`exec`，`sbrk`，`fork`。当然对这些地方做修改确实不是那么一帆风顺，需要谨慎，并且这次的代码`usertests`非常强，足够排除掉很多很多写法。

我踩过的坑有：
1. 没有遵循奥卡姆剃刀法则，采用对`uvmalloc`等函数亦步亦趋修改的方案，结果程序越改越长，对于简单一些的测例似乎是能正常跑起来，但测例比较多的时候，尤其是遇到一些特殊情况，如`exec`中的`bad`，基本失去了可维护性。这说明代码的结构还是很重要的，如果一个方案并不是很合适的话，需要耐心重构。
2. `exec`中的`bad`，似乎很容易在第一个测例中被触发，特殊情况需要考虑控制流是否是正确的。
3. `kvmcopy`和`kvmfree`这两个函数的参数传递需要注意做到页对齐了，不然基本还是会寄了。
4. 需要谨慎观察`sz`大小的变化，尤其是在`exec`函数之中，此外需要注意在`growproc`中注意空间是变大了还是变小了，对此来进行`kvmcopy`和`kvmfree`的选用。

总之这个实验我感觉还是蛮有意思的，因为近期在做科研实在没有什么大片时间去做这个实验，而实验本身的连贯性和难度却又在我的预期之外，踩了很多坑后还是自己在不参考代码，参考过思路的情况下做出来了，不过参考了思路似乎不是什么好事，虽然MIT本身是有暗示这个做法，以及部分选用xv6使用的学校也在实验文档中明确指出了这个方法，但是成就感虽然挺高（毕竟调试这件事情还是不好做的），收获比较大，但不足以让我完全相信自己的能力。

希望下次刷课是在没有干扰，并且有大片时间的情况下，不过我觉得可能这样的机会会很少。。就这样吧，这个实验真的有意思。

最后附上通关图：
```shell
== Test pte printout == 
$ make qemu-gdb
pte printout: OK (3.2s) 
== Test answers-pgtbl.txt == answers-pgtbl.txt: OK 
== Test count copyin == 
$ make qemu-gdb
count copyin: OK (1.0s) 
== Test usertests == 
$ make qemu-gdb
(148.5s) 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: OK 
Score: 66/66
```

## 附录：其他的一些代码分析：
### exec代码分析：
感觉还是有一些必要去对`exec`的代码做一些仔细的分析。
```C
int
exec(char *path, char **argv)
{
  // printf("[exec]:\n");
  uint64 origin_sz = 0;
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();
  // 与文件系统有关系，可能和同步相关联
  begin_op();
  // 文件系统访问，找到所对应的 inode pointer
  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);
  origin_sz = p->sz;
  // printf("p->sz: %p\n", p->sz);
  /*
  if(p->sz < PLIC){
    uvmunmap(p->k_pagetable, 0, PGROUNDUP(p->sz)/PGSIZE, 0);
  } else{
    uvmunmap(p->k_pagetable, 0, PLIC/PGSIZE, 0);
  }
  */
  // printf("p->sz: %p\n", p->sz);
  // kvmfree(p->k_pagetable, 0, PGROUNDUP(p->sz));
  // 从 ip 位置，找到 elf 文件的头
  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  // 检查 elf 头文件格式是否正确
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  // 根据 ELF 头文件，逐个段落进行内存上的拷贝工作
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    // 为每个段分配 memsz 大小的空间，并且将他在 pagetable 中向下继续映射
    // 分配的空间段似乎做到了连续，但实际上的这些代码段可能并不是天然就是连续的，看起来多分配一点也不是什么坏事
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    // 这边是通过文件系统 ip ，通过先前我们使用的函数 readi ，将文件偏移量从 ph.off 开始的部分进行复制，复制到 pagetable 之中
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  // 再分配两个页，其中一个打算作为用户程序的栈地址空间
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  // 设置 sz - 2*PGSIZE 用户没法访问，从而作为 guard page
  uvmclear(pagetable, sz-2*PGSIZE);
  // 让程序的栈使用第二个，即地址更高的那一个页
  sp = sz;
  // stack base 则是 sp - PGSIZE，是从 sp 向下减少
  // 这边的布局和 xv6 book 中的 figure 3-4 已经很像了
  stackbase = sp - PGSIZE;

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    // 拷贝 argv 所对应的字符串到 sp 上，最后 + 1 是为了考虑 '\0' 字符
    // copyout 是 copyin 的逆向操作，我们不再解析
    // 拷贝的 sp 位置是对的，因为我们已经先做了偏移量上的减法
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    // 在 ustack 中填入相关的地址
    ustack[argc] = sp;
  }
  // 最后一个参数则默认为 0
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  // 要把这个数组一并压入用户栈之中
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));

  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp; // initial stack pointer
  // printf("p->name: %s p->sz: %p origin_sz: %p\n", p->name, p->sz, origin_sz);
  if(origin_sz > p->sz)
    kvmfree(p->k_pagetable, PGROUNDUP(p->sz), PGROUNDUP(origin_sz));
  kvmcopy(p->pagetable, p->k_pagetable, 0, PGROUNDUP(p->sz));
  proc_freepagetable(oldpagetable, oldsz);

  if(p->pid == 1){
    vmprint(p->pagetable);
  }
  
  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  // printf("bad\n");
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```
到这边基本上就搞清楚了`exec`的原理了，相对还是比较清楚的。