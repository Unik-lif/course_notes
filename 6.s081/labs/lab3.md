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
    procinit();      // process table

    // In different branch of hartid => AP cores.
    // AP cores.
    kvminithart();    // turn on paging
}
```
### kinit函数
不得不说`xv6`实现的非常简单清楚：先初始化锁，然后开辟空闲地址空间。`end`是`xv6`代码段的终结，`PHYSTOP`是内存的终结。
```C
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```