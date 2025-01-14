## XV6的file system
重新跟着书来梳理一下文件系统的建立流程，以及解剖一下它的一些运转机制。

### 8.1 概览
最基本的展示图还是下面这样：

```
------------------
| File descriptor|
------------------
| Pathname |
-------------
| Directory |
-------------
|   Inode   |
------------
| Logging |
---------------- 
| Buffer cache |
----------------
| Disk|
-------
```
自然，按照这边的逻辑，我们应该从Disk开始，自底向上地尝试分析。

### 8.1+ 对于Disk的使用
磁盘本身必须经过一定的格式化，才可以被拿来使用。然而，我们关心的并非是驱动层上的代码，而是在驱动层之上对于disk的抽象情况。

在Makefile中内置的qemu启动脚本中，我们注意到下面这一部分：
```shell
QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

ifeq ($(LAB),net)
QEMUOPTS += -netdev user,id=net0,hostfwd=udp::$(FWDPORT)-:2000 -object filter-dump,id=net0,netdev=net0,file=packets.pcap
QEMUOPTS += -device e1000,netdev=net0,bus=pcie.0
```
可以看出启动的时候，首先需要准备一个合适的文件系统版本，否则没有办法启动，而这个fs.img，则是经过mkfs程序获得的。通过mkfs程序的解读，我们可以大概得知文件系统的布局情况。fs.img实际上就是对于物理磁盘的一种抽象，而生成fs.img的mkfs文件，则构造了这一层抽象。它这个文件，与内核中的文件是相互对应的，否则没法让内核做正确的解读。

从mkfs程序中我们可以看出，对于磁盘，xv6做了下面的抽象：将磁盘分割成一定量的block，其中nmeta个blocks用于存放变量，而剩下的blocks则存放数据。
```
| boot | superblock |   log   |  inode blocks  |       bitmap         |         data block                     |
0      1            2     2 + nlog    2 + nlog + ninodeblocks   2 + nlog + ninodeblocks + nbitmap             FSSIZE
```
其中FSSIZE反映了整个文件系统，也就是整个磁盘空间以block为单位计量得到的大小。

mkfs主要做的事情总结如下：
1. 创建一个叫做fs.img的文件，让它的大小为FSSIZE这么多的block，一开始全部写0覆盖之，每个block含有1024个字节。
2. 在1号block写下superblock信息，通过ialloc分配作为目录的inode，得到rootino，再把两个目录，也就是"."和".."，对应的信息写到这个inode中去。
3. 把users目录下的应用程序，和其他程序拿过来，分配作为文件的inode，把这些文件名对应的dirent，也就是存储文件名的inode信息，通过iappend来写到rootino上。
4. 对于应用程序的具体信息，则存储到文件名inode指向的inum位置，在inum位置进行文件实际信息到data block的iappend拷贝。
5. 知道了有多少个freeblock了，现在可以在balloc这边，向bitmap对应的blocks写入bitmap信息

其他的一些细节：
1. 一个block由多个inode组成，每隔inode都可以表达成dinode的形式，这个是on-disk inode的形态。对于inode的读写操作，本质上是对block的读写操作，只不过粒度更加细。
2. 在iappend中可以观察到direct和indirect索引的细节，对于indirect，首先分配结点，然后再在子结点继续向下分配。

到这一步，我们完成了对于文件系统的初始创建，在这边我们大概知道了inode和block可能存在的对应关系，并且已经为系统启动准备了所需要的文件结构布局和树形文件分布情况。但是，block的抽象仅仅还是第一步。
### 8.2 buffer cache layer
直接根据先前生成的disk内的文件系统来进行工作，动作还是比较慢。为了提高速度，我们引入buffer cache这一层抽象。

这一层抽象的使用方式是，一开始通过binit来进行初始化，之后利用bread和bwrite来进行读和写。在xv6中，对于我们先前做的磁盘抽象，我们并不会直接调用这一层的接口，而是必须通过buffer cache的接口，直接与更加底层的disk驱动进行交互。

bcache是一个含有30个buf大小的缓存，此外还有一个帮助索引用的head，以及用于保持同一时间仅有一个内核线程对底层磁盘进行访问的锁，其中每个buf含有其对应的具体的硬件位置信息，以及blockno，refcnt等重要信息。

这个部分相对比较清楚简单，我们讲到这边应该就可以，唯一可能需要注意的是这边对于buf的锁的应用，其中acquiresleep和holdingsleep都是和某个进程强绑定的，所以它们必须是一个对对子。
### 8.4 Logging
虽然buffer cache能够显著增强文件系统使用的便利性和速度，其对于数据缺乏防护，在崩溃的情况下，依旧很难保持一定的一致性，从而在系统的稳定上还是不太行。为此，在buffer cache之上，我们又添加了一层logging层，允许在对文件操作的同时，通过日志来记录基本的文件系统操作行为。

这样就把原本不确定的东西原子化了。