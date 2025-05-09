虚拟机的密码是123
## 实验：地址空间
在`Linux`内核中似乎是能够把内核地址空间映射到用户地址空间的，这样就能减少一些内存上拷贝的开销。

在做这个实验之前，我一般采用的方式是通过`ioremap`来做，不过对应的驱动也要多做很多次的内存拷贝，这确实会比较麻烦，因此有必要学习一下这边的一些技巧。

`kmalloc`使用的是`lowmem`部分内存空间，这部分内存空间在分配的时候是连续的，而`vmalloc`使用的是`highmem`空间，这部分内存空间则不是以连续的方式来分配的。

如果是用户来使用，我肯定更加倾向使用`kmalloc`，不过需要注意的是这个函数的返回值是虚拟地址，而且也不一定对齐了，需要简单做一个对齐工作。

### 实验环节：
#### 实验一：
首先，我们需要通过`kmalloc`来分配`NPAGES+2`这么多页，为什么如此分配，其实是为了防止分配的起始地址和结束地址不是页对齐的，我们需要做到页对齐。

`remap_pfn_range`函数是一个很关键的函数，可以把一个物理地址空间映射到一段虚拟地址空间中，虚拟地址空间在`linux`中使用`vm_area_struct`来做简单的表示。

记得做好`SetPageReserved()`，以防止分享出去的内存被交换进一步因此而被删掉。

为了更多了解实验的全貌，我们需要检查用户态的测试程序，这样才能在自己写驱动时与`userspace`交互时有一个直观的认知。
#### 实验二：
以后心情好了再做，做到实验一基本上这个流程已经搞清楚了，可以搬到自己的项目里头了。