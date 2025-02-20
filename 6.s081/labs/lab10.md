## 实验记录
做一个阉割版的mmap。

主要内容细节：

> mmap can be called in many ways, but this lab requires only a subset of its features relevant to memory-mapping a file. You can assume that addr will always be zero, meaning that the kernel should decide the virtual address at which to map the file. mmap returns that address, or 0xffffffffffffffff if it fails. length is the number of bytes to map; it might not be the same as the file's length. prot indicates whether the memory should be mapped readable, writeable, and/or executable; you can assume that prot is PROT_READ or PROT_WRITE or both. flags will be either MAP_SHARED, meaning that modifications to the mapped memory should be written back to the file, or MAP_PRIVATE, meaning that they should not. You don't have to implement any other bits in flags. fd is the open file descriptor of the file to map. You can assume offset is zero (it's the starting point in the file at which to map).


```C
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```
其中addr一直都是0，mmap来决定地址的起始位置，要么是一个正确的值，要么是0xfffffff表示失败。其中prot表示这个内存应该被映射的情况，而flag则比较特殊，表示MAP_SHARED的时候，最后的更改需要落实到文件中，而如果是MAP_PRIVATE，则说明不能落实到文件里。

好像前置条件还挺简单的，之后就跟着HINTS一步一步做着来。

### 实验步骤
简单复习了一下lazy allocations部分，看起来我们在这边也要对usertraps的异常处理部分做一些修改，主要集中在mmap分配时的调用上。

首先需要完成vma数据结构的统计和分配，具体来说，需要为每个进程单独开辟一个数据结构用来存储这一部分的信息。

对于fd的信息，观察open系统调用，可以发现相关的数据结构存放在`proc->ofile`，而不是`fdtable`中。

此外，PTE_W和PROT_READ这些并不是一一对应的，所以可能需要手动进行解析。

还有一个挺重要的地方：就是mmap这边，它输入的offset似乎一直都是0，可是usertrap这边处理的时候，是以一个Page为单位往下读的，那么偏移量其实还是需要设置起来。

此外，在not-mapped unmap这边测试的是没有被真实分配内存时的情况，这一点非常重要。我们实际上只对前两个页的空间进行了分配，第三个页并没有真的被map掉，所以只要和lazytest那边一样，把panic改一下变成continue应该就行。

fork的时候子进程把父进程的地址空间给清空了也不是什么大的事情，反正触发一下pagefault重新把东西拿回来就行了，不用太担心。

其他的一些细节面向测试就能全部搞定了。这个实验任务有一定难度，但绝对没大家网上说的那么难，我觉得最难的还得是lab9。

我是属于甲流期间冒着冷汗做的这个，脑子不清不楚感觉不能把这个时间用来想IDEA，所以就先干这个作为自己的康复疗程，也没觉得多耗费自己的脑细胞。

== Test running mmaptest == 
$ make qemu-gdb
(3.5s) 
== Test   mmaptest: mmap f == 
  mmaptest: mmap f: OK 
== Test   mmaptest: mmap private == 
  mmaptest: mmap private: OK 
== Test   mmaptest: mmap read-only == 
  mmaptest: mmap read-only: OK 
== Test   mmaptest: mmap read/write == 
  mmaptest: mmap read/write: OK 
== Test   mmaptest: mmap dirty == 
  mmaptest: mmap dirty: OK 
== Test   mmaptest: not-mapped unmap == 
  mmaptest: not-mapped unmap: OK 
== Test   mmaptest: two files == 
  mmaptest: two files: OK 
== Test   mmaptest: fork_test == 
  mmaptest: fork_test: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (105.6s) 
== Test time == 
time: OK 
Score: 140/140