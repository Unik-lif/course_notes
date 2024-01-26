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

## 实验
### 实验一：page table print
可以直接参考`freepagewalk`时的做法，很轻松就能实现。注意这边的递归可能处理起来比较麻烦，因此我使用了一个辅助函数，并且用了一个`level`变量来帮忙打印层级。

### 实验二：为每个进程分配一个kernel page table
这个实验似乎是为了性能和可用性来考虑，为了让`kernel`能够轻松地去获得用户进程的变量地址，需要为每个用户进程做一个内核地址空间，使得内核可以直接对相关变量进行访问，而不需要转换成物理地址再去处理。

