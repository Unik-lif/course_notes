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
#### step5.
对于fork系统调用的实现，特别要注意可以利用`rax`来修改返回值，这真的是太酷了。
#### step6.
很好玩，限定一下范围就可以了。
#### step7.
需要free掉一个进程的页表，这应该怎么做呢？

首先我们需要把kfree做一个简单的实现，这需要我们看一下kalloc是怎么做的。kalloc利用allocatable_physical_address函数来确定某个物理地址是否是用于被kalloc进行分配的区域，并且还要看当前这个物理地址是否被拿去分配了。

因此kfree的实现首先要查看给出的地址是否是用于分配的区域，其次还要检查这个区域是否被多个进程拿去使用。如果该物理地址区域仅仅被一个进程所占用，那么他对应的refcount应该是1。仅仅在这种情况下，我们才有把它free掉的价值。
```
0x000000 R6243SS.33K4443475553353335777737CCCCCCCCCCCCC77CCCCCCCCCC555344
0x040000 KKKKKKKKKKKKKKKKKKKKKKKKKK47444CCC777777744444488C8888884999C9CK
0x080000 CCCCCCCCCCCCCCCC3933333937337333RRRRRRRRRRRRRRRRRRRRRRRRCRRRRRRR
0x0C0000 RRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRR
0x100000 33333733339333333333933333343333333444444447S4444434A44444444444
0x140000 447444444444444444444444444444444444.4444AA7AA44111111144AA44444
0x180000 44C6667666763A4822222222444.22C4444442722C2222447777777472742222
0x1C0000 2272777277722227777777777777777722222222222222222222444344444444

                      VIRTUAL ADDRESS SPACE FOR 12
0x000000  6243SS.33K4443475553353335777737CCCCCCCCCCCCC77CCCCCCCCCC555344
0x040000 KKKKKKKKKKKKKKKKKKKKKKKKKK47444CCC777777744444488C8888884999C9CK
0x080000 CCCCCCCCCCCCCCCC3933333937337333RRRRRRRRRRRRRRRRRRRRRRRRCRRRRRRR
0x0C0000 RRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRR
0x100000 SSCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
0x140000  
0x180000 
0x1C0000 
0x200000 
0x240000 
0x280000 
0x2C0000                                                                C


PANIC: Kernel page fault for 0x0 (read missing page, rip=0x41db1)!
```
有一点遗憾，这个东西为什么会访问0，我并没有搞清楚。

第二个问题，PAGE TABLE ERROR: 5f000: kernel allocated page mapped for user (pid 2)
### 一些技巧与别的：
利用`grep -rn "kernel_pagetable"`来快速查找，类似于开通了`ack`功能的`vim`。

代码写的不错，希望未来能够抽个时间通读一遍。