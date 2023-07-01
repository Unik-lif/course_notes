## WeensyOS
前段时间去通关rCore了，这玩意儿一直给我鸽者，这其实很不好。

重新开始玩吧！
### 概要：
WeensyOS支持3MB大小的虚拟地址，运行在2MB大小的物理地址之上，是一个很小的操作系统。
### general question：
1. What is the disadvantage of having an identity mapping between virtual and physical memory? What is the purpose of mapping virtual memory addresses to different physical addresses?

这个问题很简单，如果只能做恒等映射，至少会造成复用性很弱，进程之间可能无法共享自己的页。而且很容易出现内存碎片的问题，使用起来很困难。

2. How does a new page table get prepared for a new process? Talk both about processes born from fork() and those not born from fork().

如果是通过fork得到的，我们维持原本的就可以了。如果不是通过fork得到的，我们需要装载相关的`elf`文件。

3. During the normal execution of the process, how does the process refer to its memory? Go through an example of translating a virtual memory address to an actual physical memory address.

4. Upon exiting, what kind of resources have to be cleaned up and freed?

### p-allocator.cc
```
----------------------- 0
|        code         |      text, data, bss segments.
----------------------- 
|        heap         |      heap.
----------------------- <----- heap top
|                     |
|                     |
|                     |
----------------------- <----- stack bottom. rsp.
|        stack        |      stack.
----------------------- 
|                     |
|                     |
----------------------- 64KB
```
内存的布局情况：有一说一这个图确实画的很漂亮。
```
INITIAL PHYSICAL MEMORY LAYOUT

  +-------------- Base Memory --------------+
  v                                         v
 +-----+--------------------+----------------+--------------------+---------/
 |     | Kernel      Kernel |       :    I/O | App 1        App 1 | App 2
 |     | Code + Data  Stack |  ...  : Memory | Code + Data  Stack | Code ...
 +-----+--------------------+----------------+--------------------+---------/
 0  0x40000              0x80000 0xA0000 0x100000             0x140000
                                             ^
                                             | \___ PROC_SIZE ___/
                                      PROC_START_ADDR
```
在WeensyOS中，内核和全部的进程都共享一个地址空间。

在这边阅读材料提到了`protected control transfer`，说白了这个东西其实和我们的陷入机制没有什么太大的区别。
### 任务
主要需要在`kernel.cc`文件中做一些简要的修改。
#### step2.
一开始在没有装载程序的时候就把地址空间映射好了，这是蠢的，需要修改。因此需要在`load`程序的时候同步这个映射。

在`load`中我们可以看到有代码段、数据段、堆等，堆的map需要在系统调用开始时再进行，前两个则是需要看到writable的flag是否真实存在。
#### step3.
需要注意vmiter的妙用，可以方便我们得到pa。
### 一些技巧与别的：
利用`grep -rn "kernel_pagetable"`来快速查找，类似于开通了`ack`功能的`vim`。

代码写的不错，希望未来能够抽个时间通读一遍。