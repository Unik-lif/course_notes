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
### 8.6 Logging Code
使用Logging的方法看起来还是比较清楚的，当系统的其他操作，比如部分涉及文件系统的系统调用，比如EXEC指令执行时，首先会跑一下begin_op，在完成了操作之后，则会跑一下end_op。

在更加高级的使用方式下，bwrite这个接口将会被替换，使用方式将会是下面的流程。
```C
begin_op();
...
bp = bread(...);
bp->data[...] = ...;
log_write(bp);
...
end_op();
```
首先研究一下begin_op和end_op，作为对应的一组操作，其中包含这Logging的一些细节。
```C
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      // 需要把log.lock释放出去，不能抱着它睡，不然永远醒不了
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```
从begin_op中可以看出，首先需要获得log对应的锁，同一时期，仅存在一个log的请求，因此这边对于文件系统的更新可以说是单线程的。outstanding则用来记录，当前在预留的log space空间中，system calls存在的个数。由于在xv6中的logging系统，支持group commit，单次的logging commit可以服务多个不同的系统调用，这样可以大大减少系统调用陷入到与硬件交互的总共次数，从而提升性能。在begin_op中，首先检查当前这个log系统是否正在上传东西，为了同步，如果有，必须先睡眠一会儿。第二个检查则是检查空间大小是否足够。这边还假设，每个system call都能用完自己能用完的最大的文件系统更新对应的blocks数。log.lh.n的值反映的是接下来这个log操作尚还没有完成的block总数，因为完成之后，这些东西就会被删光光，而block[LOGSIZE]数组将会存放对应的blockno位置。

实际上我觉得log.lh.n反映的恰好是过去的log.outstanding已经被写道log中的结果，其实就是log_write完成之后的样子。

此外，outstanding表示正在跑，也就是完全还没有处理的，先前进来的system call所对应的情况。log.outstanding表示在begin_op开始前，已有的system call事物，而begin_op又标志一个新的system call事物的开始，因此这边会有+1的过程。

在我们完成了begin_op之后，既然已经通过bread得到了现有的数据（bread不使用log来做接口封装感觉是一种很自然的事情，因为不会涉及到数据的破坏），接下来就可以先把log信息写道log的头里，之后就可以准备真正落实相关的写操作。
```C
// 在调用log_write之前，在struct buf *b对应的信息已经在内存中完成了撰写
// 在log中写下信息: b块被写过，动过手脚了
void
log_write(struct buf *b)
{
  int i;

  // 最多一个transaction只能提交LOGSIZE这么大的数目
  // log.size则是一开始superblock在初始化的时候设置的，专门分配给log的空间，自然也是不能超过这个空间
  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  // outstanding如果比1还小，说明或许就没有log_write的必要，因为所有的log.lh.n其实已经写好了，不需要再加了
  // 或者也可以这么理解：我们规定了log_write的使用必然是在begin_op之后，而且这个使用还是单线程的，所以不存在log.outstanding小于1的可能性
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  // 先找一下目前已经记录的log.lh中有没有恰好和自己同一个blockno的，这样就可以节省一些操作
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion，这样可以把多个写操作合并在一起，这里是说b锁对应的blockno将会被写，具体怎么写，这边不涉及
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    // 防止给驱逐除去了，不然的话又得读一下磁盘
    // 还有一个好处在于，保证这个log不会被驱逐，因此能够保证log信息的更新是原子的，即要么一口气更新完，要么就啥都没有更新
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```
我们在这里可以发现，log.outstanding的最终结局就是变成log.lh.n中的block，被压到log.header之中。因此实际上这边应该是有三个阶段。
```
system call -> outstanding -> log.lh.n
```
我们再从同步的角度来看一下这件事情，假设同时有两个操作文件系统的system call，它们一定会有一个执行的先后顺序，因为log.lock的存在。我们不管之后的顺序是什么样子的，但一件事情很明确，就是log.outstanding的值它确实可以不只是为1，因为当begin_op结束之后，针对log.lock锁的持有就已经结束了。两个system call最后都会跑一个end_op，因此最后一次的end_op，一定会出现log.outstanding为0的情况，在这种情况下，log.commiting就可以设置为1了。当log.commiting设置为1时，begin_op操作就被锁死，用来避免对于正在进行的committing行为的干扰。

之后，会通过end_op来落实已经被储备在log.lh.n中的块。
```C
// called at the end of each FS system call. 也因此，我们可以对于log.outstanding减去一个一
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    // 感觉还是很合理的，如果前面有begin_op因为空间不够被堵住，现在它可以跑了
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    // 更关键的原因时begin_op在do_commit的时候会被锁住动不了
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```
在了解了同步的流程之后，我们集中来看commit的具体流程：
```C
static void
commit()
{
  // log.lh.n即为想要落实更新的部分
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log，单纯把磁盘中的log.lh.n更新为0就可以了，其他的数据当作废数据，别用就行。
  }
}

// Copy modified blocks from cache to log.
// 需要注意的是，log_write步骤发生之前，我们就已经在把buf block从磁盘中读出来了，并且对其做了一定程度上的修改。
// 而在这边，我们在to这边的环节，是把磁盘信息读到缓存里，而第二个from则不一样，由于先前我们bpin过，它会自动索引到修改过的缓存位置，而这个缓存信息并没有同步修改到磁盘中
// 这一步把数据块写到了LOG的位置，之后要根据LOG HEADER的索引来把这些LOG真实地INSTALL下来
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    // log 块的第一个位置，也就是log.start位置似乎被分配给了log header block
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    // 这一步是把缓存block写到log block对应的缓存中，但是log block对应的缓存还需要做一下同步
    memmove(to->data, from->data, BSIZE);
    // 保证能够落实到磁盘中，这一步就从log block走到disk中去了
    bwrite(to);  // write the log
    // from部分可以丢掉了，先前是故意bpin的，to部分也可以消除掉，同步工作结束之后，就不需要这一部分缓存了
    brelse(from);
    brelse(to);
  }
}

// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
// 在这一步结束之后，我们的log才真正地落实到了磁盘中
static void
write_head(void)
{
  // 第一个位置好像专门送给了log header，但是为什么要把log header存进去，或许是为了让disk能够好好索引？
  // 确实，log header存放了这些数据块之后要被INSTALL的具体位置，没有LOG HEADER，就没有办法真实地在之后让这些LOG写道指定的位置
  // LOG BLOCK的第一个BLOCK恰好就是专门用来存放LOG HEADER的位置
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}

// Copy committed blocks from log to their home location
static void
install_trans(int recovering)
{
  int tail;
  // 根据log.lh.block[tail]信息，来锁定log真正要写入到的block位置，首先更新cache，然后再把cache落实到磁盘中
  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    // 最后目的都是保证把dbuf和lbuf减小到0，这样才能彻底清除从缓存，落实到disk中
    // 然后recovering为0时，先前log.lh.block[tail]应该已经引用了一次
    // 而启动的时候，则没有多引用，我觉得这边的值可能是一个修正值，我们完全可以打印来看，说实在的这个位置应该算是脏活
    if(recovering == 0)
      bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}

```
完成了commit操作之后，直接committing值已经可以设置为0了，然后唤醒先前因为committing为1时被迫陷入睡眠状态的请求就行了。

那么，如果程序真的崩溃了，我们如何才能通过使用Logging机制来帮助我们恢复状态呢？这需要我们更加关注一些涉及恢复的程序。按照我们的使用习惯，恢复相关文件系统的函数会在系统重启时得到使用。这似乎很容易找到，在userinit程序调用了initcode中的程序后，通过scheduler我们应该很快就能切换到这边的forkret，而第一次执行forkret程序，我们就可以跑起来对于文件系统的恢复程序。
```C
// A fork child's very first scheduling by scheduler()
// will swtch to forkret.
void
forkret(void)
{
  static int first = 1;

  // Still holding p->lock from scheduler.
  release(&myproc()->lock);

  if (first) {
    // File system initialization must be run in the context of a
    // regular process (e.g., because it calls sleep), and thus cannot
    // be run from main().
    first = 0;
    fsinit(ROOTDEV);
  }

  usertrapret();
}

// Init fs
void
fsinit(int dev) {
  // 首先读了一下这个superblock，这个superblock应该是根据fs.img来创建的，应该会包含一些基本的修改情况
  readsb(dev, &sb);
  if(sb.magic != FSMAGIC)
    panic("invalid file system");
  initlog(dev, &sb);
}

// 初始化log系统的时候，就应该来查看一下是否有一些log是崩溃的时候记下来的，而目前却没有落实下来
void
initlog(int dev, struct superblock *sb)
{
  if (sizeof(struct logheader) >= BSIZE)
    panic("initlog: too big logheader");

  initlock(&log.lock, "log");
  log.start = sb->logstart;
  log.size = sb->nlog;
  log.dev = dev;
  recover_from_log();
}

static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}

// Read the log header from disk into the in-memory log header
static void
read_head(void)
{
  // 从disk中，读出来到buf，这边读的是log header
  struct buf *buf = bread(log.dev, log.start);
  // 检查log header中是否有一些数，最重要的是这个log.lh.n
  struct logheader *lh = (struct logheader *) (buf->data);
  int i;
  log.lh.n = lh->n;
  // 更新log.lh.block信息，这个信息在崩溃之后会从内存中消失，现在我们是把它复原出来
  for (i = 0; i < log.lh.n; i++) {
    log.lh.block[i] = lh->block[i];
  }
  brelse(buf);
}
```
好像还怪简单的，不过存在脏活，log.lh.block[tail]在非recovering时可能引用次数到了2，因此可能需要用bunpin来修正一下，其他的好像也就那样。
### 8.7 Block allocator
这个部分的章节似乎有点突兀，我们在分析了balloc的基本功能之后，来看看balloc这样的东西是在何种情况下被调用。

可以看到，balloc仅在函数bmap中得到使用：看起来这个函数和inode相关，但到目前，我们尚不清除inode具体的用途，所以我们可以暂时不用管这个，这是又一层高级抽象。
```C
// Allocate a zeroed disk block.
static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;

  bp = 0;
  // BPB: bitmap per block, every block has 1024 bytes, each bytes has 8 bits, so BPB is 8192.
  for(b = 0; b < sb.size; b += BPB){
    // 需要读取bitmap中反映第b个block块是否空闲的这一个bit的信息，就需要以block的粒度把这个bitmap位置对应的整个block读出来
    // 之后再修改这个bitmap，因为操作必须是以block为单位的，所以这边会稍微麻烦一些
    // 此外，由于b的步长为BPB，所以相邻的b所对应的block是不一样的
    
    // 因此这边是对于每个bitmap块，的每一位这样双重遍历搜索下来
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      // 由于存储单位是以8为倍数的，因此得先找到对应的字节，然后再用m去找对应的那一个位，用来确认是否是空的
      // 确认是空的话，设置为1，就是或上m
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        // 毕竟对bp做了一些更新，在这边先修改log header，之后在end_op落实下来
        log_write(bp);
        brelse(bp);
        // 对于 b + bi 所对应的 blockno ，记得写它们为0
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}

// Free a disk block.
static void
bfree(int dev, uint b)
{
  struct buf *bp;
  int bi, m;

  // 反向操作，感觉比较好理解了，这边的 b 就是上面返回的 b + bi
  bp = bread(dev, BBLOCK(b, sb));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic("freeing free block");
  bp->data[bi/8] &= ~m;
  // 这边主要还是更新信息哈哈哈哈，修改一下log header，等者在end_op的时候落实下来
  log_write(bp);
  brelse(bp);
}
```
疑似这边的balloc是分配了一定的磁盘空间给某个位置，且这一部分空间已经被很好的设置为0了，对应的bitmap也分配出去了。那么，到底是谁或者哪些函数在使用这个功能呢？我们接下来就可以考虑研究一下inode的表现了。
### 8.8 Inode 层
立于log层之上的是inode，看起来inode的主要使用是通过balloc和bfree动态分配对应的block空间，在这个过程中通过log_write来调用log相关的接口。

inode的两幅面孔：
1. 在磁盘中包含文件大小和data block numbers的具体信息，用来保存文件的基本信息（可以视作文件的属性栏）
2. 在内存中的inode，其包含磁盘中的inode信息，以及内核中与之关联的额外信息


第一类inode情况，看起来长得很好看，一共NDIRECT个直接可以连接的data block位置，还有一个位置负责间接连接，这样一来文件的大小可以是很大。
```C
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```
第二类是在memory中的inode，它们相对而言会更加活跃：其中包含这dinode的一部分信息，还有一些dev，inum，ref等额外信息。
```C
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count，当前有多少活跃的C指针指向它
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```
其中iput和iget分别是把inode信息写回到disk，和从disk读取出来到memory中的两个接口，很自然它们会改变struct inode数据结构的ref数据。