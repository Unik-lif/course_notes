## 学习PPT
物理内存是可以以`alloc_pages`的方式进行分配的，以便满足特殊的`contigous memory allocations`需求。

在这边简单介绍了`buddy system`分配大物理空间，`slab allocator`分配小物理空间的`Linux`基本使用内存分配器手段。

这一套机制在很多开源小型`OS`中均有使用，比如`linux svsm`，`2.6`版本的`Linux`似乎已经支持了这个功能。

值得注意的是，`slab allocator`中使用了缓存优化技巧，即染色技巧。

