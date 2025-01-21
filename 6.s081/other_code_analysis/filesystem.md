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
### 8.9 Inode 代码
为了分配Inode，我们使用接口ialloc函数：
```C
// Allocate an inode on device dev.
// Mark it as allocated by  giving it type type.
// Returns an unlocked but allocated and referenced inode.
// type表示的是，分配的是文件还是一个目录
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;
  // 这边从1开始疑似是为了和mkfs这个工具维持一致，或许0的位置是专门为根目录留出来的，当然也存疑
  for(inum = 1; inum < sb.ninodes; inum++){
    // 首先确认究竟是哪一个block包含序号为inum的inode
    bp = bread(dev, IBLOCK(inum, sb));
    // 找到inum指定的inode结构
    // 这边看起来是向下顺序分配着的
    dip = (struct dinode*)bp->data + inum%IPB;
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      // 希望之后对bp内的dip位置真实做点修改，在log_write把东西写到log_header里，等待end_op更新过来
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp);
      // 继续研究iget的功能
      // iget这边更像是保证在icache已经有了这个inode的信息，但是确实是unlocked状态的，需要在ilock环节保证信息的合理性
      return iget(dev, inum);
    }
    // 如果type不等于0，说明这个inode已经被分配了类型了，就不需要再次在磁盘block中更新了
    brelse(bp);
  }
  panic("ialloc: no inodes");
}

// Find the inode with number inum on device dev
// and return the in-memory copy. Does not lock
// the inode and does not read it from disk.
// iget最后似乎会返回一个in-memory的拷贝所对应的指针
// 不同环节的iget最后返回的指针信息地址信息都是一样的，只不过可以有多个不同的拷贝副本
// 这也为之后ilock对于inode的exclussive access做了一些准备
static struct inode*
iget(uint dev, uint inum)
{
  struct inode *ip, *empty;

  acquire(&icache.lock);

  // Is the inode already cached?
  // 首先检查当前的这个inode是否已经被存放、管理在icache之中了
  // icache和bcache似乎是不同的机制
  empty = 0;
  for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++){
    // 这个inode对应的位置，已经在icache中已经存在了
    // 似乎并不意味着ialloc调用的iget就一定不会进入这个分支，因为总体上可以视作存放dev，inum的位置算是固定的
    // 对于disk-inode信息，很有可能还没有拷贝进来？
    if(ip->ref > 0 && ip->dev == dev && ip->inum == inum){
      ip->ref++;
      release(&icache.lock);
      return ip;
    }
    // 尝试记录第一个empty slot的位置，这个将会分配给新来的
    // 存在下面说的recycle的可能性
    if(empty == 0 && ip->ref == 0)    // Remember empty slot.
      empty = ip;
  }

  // Recycle an inode cache entry.
  if(empty == 0)
    panic("iget: no inodes");

  // 如果在icache中没有我们dev inum位置对应的
  ip = empty;
  ip->dev = dev;
  ip->inum = inum;
  ip->ref = 1;
  ip->valid = 0;
  release(&icache.lock);

  return ip;
}
```
在上面的分配和利用iget在icache中注册，并返回对应的inode pointer的行为很有可能并没有存放除了dev和inum等必要的信息指标，正如ialloc环节所说的，返回的是一个指向unlocked状态的inode。为了让其真正能够发挥功效，我们需要在ilock等函数中，正儿八经地把disk inode的信息拷贝过来给in-memory的inode使用。

为此，我们重点关注一下这边的ilock对应的函数细节：
```C
// Lock the given inode.
// Reads the inode from disk if necessary.
void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic("ilock");
  
  // 把这个inode指针对应的lock从这个时候就一直攥在手上
  // 其他的inode指针，此时就只能卡在这里睡觉，直到含有ilock的行为自己通过iunlock释放开来
  acquiresleep(&ip->lock);

  // 其他的数据不可信，但是ip对应的dev和inum是可信的，根据这个足够能够通过bread来找到对应的block pointer了，然后进一步找到inode对应的位置
  if(ip->valid == 0){
    bp = bread(ip->dev, IBLOCK(ip->inum, sb));
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    // 很基本的拷贝
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```
那么紧接着观察iunlock的情况，这会比较合适：
```C
// Unlock the given inode.
void
iunlock(struct inode *ip)
{
  // holdingsleep: 保证当前进程确实是因为ip->lock锁住的，而且确实和上锁的那个进程是同一个
  if(ip == 0 || !holdingsleep(&ip->lock) || ip->ref < 1)
    panic("iunlock");

  releasesleep(&ip->lock);
}
```
与iget对应的操作是iput：
```C
// Drop a reference to an in-memory inode.
// If that was the last reference, the inode cache entry can
// be recycled.
// If that was the last reference and the inode has no links
// to it, free the inode (and its content) on disk.
// All calls to iput() must be inside a transaction in
// case it has to free the inode.
void
iput(struct inode *ip)
{
  acquire(&icache.lock);
  // 如果当前只有一个引用（毕竟iget和iput是配合使用过的），并且当前甚至还没有文件是使用它这个inode的
  if(ip->ref == 1 && ip->valid && ip->nlink == 0){
    // inode has no links and no other references: truncate and free.

    // ip->ref == 1 means no other process can have ip locked,
    // so this acquiresleep() won't block (or deadlock).
    acquiresleep(&ip->lock);

    release(&icache.lock);

    // 困惑的地方：为什么不在itrunc这边先别急着iupdate?明明东西还没改完？
    // 可能是为了代码的维护性更好吧？
    itrunc(ip);
    ip->type = 0;
    iupdate(ip);
    // 这个是in-memory的ip特有的，不需要更新到disk中去
    ip->valid = 0;

    releasesleep(&ip->lock);

    acquire(&icache.lock);
  }

  ip->ref--;
  release(&icache.lock);
}

// Truncate inode (discard contents).
// Caller must hold ip->lock.
// 既然没有人需要这个inode了，那么inode对应的文件内容也可以删除掉了
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;
  // 看起来就是通过bfree的接口来把信息清除掉，先清除的是NDIRECT部分
  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }
  // 查看在indirect这边是否有需要被清除掉的
  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}

// Copy a modified in-memory inode to disk.
// Must be called after every change to an ip->xxx field
// that lives on disk, since i-node cache is write-through.
// Caller must hold ip->lock.
// 需要动底下的信息，包括icache部分，全部都要改
void
iupdate(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  bp = bread(ip->dev, IBLOCK(ip->inum, sb));
  dip = (struct dinode*)bp->data + ip->inum%IPB;
  dip->type = ip->type;
  dip->major = ip->major;
  dip->minor = ip->minor;
  dip->nlink = ip->nlink;
  dip->size = ip->size;
  memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
  log_write(bp);
  // 既然对bp做了一些修改，那么现在还是像以前一样落实
  brelse(bp);
}
```
到这里就基本把inode梳理完了，看起来还是比较清楚的。
### 8.10 inode 内容
inode通过addrs来存放自身内容，其中12个是直接连接的data block块位置，剩下一个是间接的连接data block块位置。对于间接连接，其通过一整个block来存储更多地址，因此正好有1024个字节，所以有256个data block地址可以存放其中。

我们首先对bmap来进行解读：
```C
// Return the disk block address of the nth block in inode ip.
// If there is no such block, bmap allocates one.
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  // bn 表示当前还是为空的那个block的序号
  // 如果 bn 要小于NDIRECT的话，就按照直接连接的方式来做就可以了
  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  // 如果 bn 的数要大于NDIRECT，就可以走下面这条路
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      // 首先为INDIRECT分配一个块，之后则根据bn的值，在INDIRECT后再连接对应的数据块
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    // bp对应的是INDIRECT块，对其内容做解析
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    // bn对应的位置如果恰好没有被分配，则分配一下就好了
    // 对bp这个块做了修改，因此需要log_write写一次
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  
  // 就大小上不能超过INDIRCT + NDIRCT
  panic("bmap: out of range");
}
```
到这一步已经很接近更上面的文件系统了，尤其是考虑到bmap是为readi和writei服务的这一点，我们查看一下这两个函数的情况：
```C
// Read data from inode.
// Caller must hold ip->lock.
// If user_dst==1, then dst is a user virtual address;
// otherwise, dst is a kernel address.
int
readi(struct inode *ip, int user_dst, uint64 dst, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  // off 表示起始位置，n 表示向下读的字节数目，不能超过下面的范围，感觉很正常
  // 自然，n 不可以是负数
  if(off > ip->size || off + n < off)
    return 0;
  // 如果 n 可能会导致读到最后，那么就读到最后，不要超过文件的总大小
  if(off + n > ip->size)
    n = ip->size - off;

  // 反正就是全部读完呗
  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
    // ip 表示要读取的 inode pointer
    // 我们首先找到 inode content 的起始位置
    // 可以根据 bmap 来轻松找到 off 对应的起始 data block 位置，会返回一个地址号，直接读取就可以了
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    // 这表示的应该是在一个 block 内，我们要读的字节数
    m = min(n - tot, BSIZE - off%BSIZE);
    // 总之就是修改了 dst 的信息，但是 dst 是在内存上的，所以不影响
    // 如果返回的是 -1，则失败了，得退出来
    if(either_copyout(user_dst, dst, bp->data + (off % BSIZE), m) == -1) {
      brelse(bp);
      tot = -1;
      break;
    }
    // 对于 bp 并没有做改动，但是现在我们不需要了，所以完全可以 release
    brelse(bp);
  }
  return tot;
}

// Write data to inode.
// Caller must hold ip->lock.
// If user_src==1, then src is a user virtual address;
// otherwise, src is a kernel address.
// Returns the number of bytes successfully written.
// If the return value is less than the requested n,
// there was an error of some kind.
int
writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  // 经典检查
  if(off > ip->size || off + n < off)
    return -1;
  // 不能写的超过了可以存储的范畴，最大范畴就是MAXFILE * BSIZE
  // 而MAXFILE的大小则是经典的直接连接和间接连接的的 data block
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    // 到这里其实还都是一样的
    if(either_copyin(bp->data + (off % BSIZE), user_src, src, m) == -1) {
      brelse(bp);
      break;
    }
    // 但是要写到bp里头，bp对应的disk，不是内存，所以要在log_write里头更新
    log_write(bp);
    brelse(bp);
  }
  // 需要更新ip->size大小，off在之后会不停地加
  if(off > ip->size)
    ip->size = off;

  // write the i-node back to disk even if the size didn't change
  // because the loop above might have called bmap() and added a new
  // block to ip->addrs[].
  iupdate(ip);

  return tot;
}
```
这俩感觉还是比较好理解的。到现在，我们应该只差最上头的pathname，以及filedescriptor相关的东西，应该明天肯定能够完结这个令人头秃的章节了。
### 8.11 Directory Layer
文件夹类型层，本身自然也是一个文件。它对应的inode是T_DIR类型，并且其对应的data block存放着各个entries的名称。其中，dirent的类型如下所示：
```C
// Directory is a file containing a sequence of dirent structures.
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```
每一个dirent中包含一个自身inode对应的号，以及自己的名字，一共占据16个字节。为了对这些directory进行检索，我们需要调用一些文件接口：
```C
// Look for a directory entry in a directory.
// If found, set *poff to byte offset of entry.
// 返回dirent下某个含有name作为文件名对应的inode，以及返回其在父文件夹下的偏移地址
struct inode*
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de;

  // 首先保证 dp->type 的类型是T_DIR
  if(dp->type != T_DIR)
    panic("dirlookup not DIR");
  // 对当前这个 dirent 的数据块们（也就是自动从addrs部分开始）进行遍历，为此我们需要调用readi来帮助我们，这就是玩抽象的好处
  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlookup read");
    // 读到的这个dirent entry对应的inum为0，那应该就是还没初始化，因为inum的号应该是从1开始的
    if(de.inum == 0)
      continue;
    // 看看名字是否是相符的
    if(namecmp(name, de.name) == 0){
      // entry matches path element
      // 如果存在返回地址的需求，那就返回offset呗
      if(poff)
        *poff = off;
      inum = de.inum;
      // 这是经典inode获取的起手式
      return iget(dp->dev, inum);
    }
  }

  return 0;
}
```
书本中说，这个函数的行为促使了iget返回的是一个unlocked的inode。原因在于，调用dirlookup的时候，由于访问了dp的数据，一般往往是需要通过ilock方式锁定dp的（读取inode信息就是需要先给他上锁，不然连磁盘中的数据信息都同步不过来，像这边就需要磁盘中的数据信息），因此为了防止可能出现的重复上锁，也就是访问文件"."时重复上锁了，这边自然需要做到一个unlocked状态。

而dirlookup似乎被dirlink高强度调用了，dirlink主要的工作是，把一个给定的名称和inode number作为信息，生成一个新的dirent entry，写到directory对应的数据中。
```C
// Write a new directory entry (name, inum) into the directory dp.
int
dirlink(struct inode *dp, char *name, uint inum)
{
  int off;
  struct dirent de;
  struct inode *ip;

  // Check that name is not present.
  // 通过lookup方式检查是否已经存在了，如果不为0就说明已经存在，我们可以把ip送回去，因为读出来的时候已经有iget这个操作了，要逆操作一下
  if((ip = dirlookup(dp, name, 0)) != 0){
    iput(ip);
    return -1;
  }

  // Look for an empty dirent.
  // 老方法找到一个空的inode位置，这样我们可以把新的信息，name和inum写进去
  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlink read");
    if(de.inum == 0)
      break;
  }
  // 很经典地准备写进去，现在暂时先存放到de中
  strncpy(de.name, name, DIRSIZ);
  de.inum = inum;
  // 那我问你？你现在新的entry只是在这个函数中临时有效，你觉得是不是需要写到父文件夹的data block位置呢？
  if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
    panic("dirlink");

  return 0;
}
```
看起来还是很好理解的，只要慢慢读代码，一切都会好起来的。
### 8.12 Path Names
我们先前已经可以通过directory来包装了，那么下一步就是文件路径的封装。本质上，文件路径就是对于每一层都调用一次dirlookup，感觉挺真实的。

处在路径检索核心的是下面这两个函数：
```C
// 本质上都是对path的简单解析工作
struct inode*
namei(char *path)
{
  char name[DIRSIZ];
  return namex(path, 0, name);
}

struct inode*
nameiparent(char *path, char *name)
{
  return namex(path, 1, name);
}

// Look up and return the inode for a path name.
// If parent != 0, return the inode for the parent and copy the final
// path element into name, which must have room for DIRSIZ bytes.
// Must be called inside a transaction since it calls iput().
static struct inode*
namex(char *path, int nameiparent, char *name)
{
  struct inode *ip, *next;
  // 这个是根目录，直接走向ROOTINO就好了
  if(*path == '/')
    ip = iget(ROOTDEV, ROOTINO);
  else
    // 否则，则为当前进程所在的目录对应的inode，增加一次引用
    // 这个对应的inode应该是根上的inode吧？
    ip = idup(myproc()->cwd);

  // 逐个向下走，path表示儿子和孙子，name表示父亲目录
  while((path = skipelem(path, name)) != 0){
    // 读当前ip对应的inode，以方便向下锁定path
    ilock(ip);
    // 类型肯定得是T_DIR啊，不然这辈子有了
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    // nameiparent要么是1,要么是0,如果是1,并且路径啥都没写
    // nameiparent负责返回最后一个元素的父亲目录
    // 遍历到path为空的情况下，也就是path中不存在父亲目录了
    // nameiparent为1,则返回父亲目录，恰好是这个ip
    if(nameiparent && *path == '\0'){
      // Stop one level early.
      iunlock(ip);
      return ip;
    }
    // next表示的是通过dirlookup找到在name下，儿子目录对应的inode
    // 返回0说明找不到
    if((next = dirlookup(ip, name, 0)) == 0){
      iunlockput(ip);
      return 0;
    }
    // 发现ip始终是考虑向下遍历的
    iunlockput(ip);
    // 现在ip指向儿子目录了
    ip = next;
  }
  // 感觉这是不应该涉足的地方，毕竟前面有啊，有一定概率是路径出现错误了
  // 什么错搞不清楚，前面明明有类似分支啊。。。
  // 这种情况确实应该返回0倒是
  if(nameiparent){
    iput(ip);
    return 0;
  }
  // 按照常理，应该正好读到一个文件，当然是在nameiparent为0的情况下，为1的情况在前面被覆盖了。
  return ip;
}
```
### 8.13 File Descriptor Layer
相较于path来说，我们可以通过更加抽象的File Descriptor Layer来尝试描述这件事情。FD可以用于描述文件，console，和很多东西，是一个相对通用的东西。
```C
// Allocate a file structure.
struct file*
filealloc(void)
{
  struct file *f;

  acquire(&ftable.lock);
  // 以file作为粒度向下进行扫描，这边是逐个文件的扫描
  for(f = ftable.file; f < ftable.file + NFILE; f++){
    // 找一个还没有用上的
    if(f->ref == 0){
      f->ref = 1;
      release(&ftable.lock);
      return f;
    }
  }
  // 满了，没法分配
  release(&ftable.lock);
  return 0;
}

// Increment ref count for file f.
struct file*
filedup(struct file *f)
{
  acquire(&ftable.lock);
  if(f->ref < 1)
    panic("filedup");
  f->ref++;
  release(&ftable.lock);
  return f;
}

// Close file f.  (Decrement ref count, close when reaches 0.)
void
fileclose(struct file *f)
{
  struct file ff;

  acquire(&ftable.lock);
  // 尝试关掉一个file对应的file descriptor
  if(f->ref < 1)
    panic("fileclose");
  if(--f->ref > 0){
    release(&ftable.lock);
    return;
  }
  // 如果此时f->ref已经可以被减小到0了
  // 先把原来用的这个f的信息拷贝到ff之中
  ff = *f;
  f->ref = 0;
  f->type = FD_NONE;
  release(&ftable.lock);

  // 如果原来的ff对应的类型是FD_PIPE
  if(ff.type == FD_PIPE){
    // 那说明我们应该以关闭PIPE的方式来关闭它
    pipeclose(ff.pipe, ff.writable);
  } else if(ff.type == FD_INODE || ff.type == FD_DEVICE){
    // 否则，则是持久性缓存类型的文件
    // 这说明在关闭文件之前，我们确实已经对f->ref做了修改，改成了0，而这个信息需要同步到磁盘之中
    begin_op();
    iput(ff.ip);
    end_op();
  }
}
```
通过filealloc来对存储了全局fd信息的ftable进行扫描，行为模式非常简单。filedup好像只是增加了某个file descriptor的引用数目。fileclose的功能亦非常清楚简单。
```C
// 感觉这个filestat非常合理
// 仅支持会存放在磁盘中的inode结构来进行操作，如果类型不是FD_INODE或者FD_DEVICE就不行了
// Get metadata about file f.
// addr is a user virtual address, pointing to a struct stat.
int
filestat(struct file *f, uint64 addr)
{
  struct proc *p = myproc();
  struct stat st;
  
  if(f->type == FD_INODE || f->type == FD_DEVICE){
    ilock(f->ip);
    stati(f->ip, &st);
    iunlock(f->ip);
    if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
      return -1;
    return 0;
  }
  return -1;
}
// Read from file f.
// addr is a user virtual address.
int
fileread(struct file *f, uint64 addr, int n)
{
  int r = 0;

  // 检查这个文件是否可读
  if(f->readable == 0)
    return -1;

  // 检查文件类型是否为PIPE类型
  if(f->type == FD_PIPE){
    // pipe有自己的一套读法
    r = piperead(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    // 设备也有自己的一套读法
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(1, addr, n);
  } else if(f->type == FD_INODE){
    // inode 的读法：先从disk中读出来inode信息，之后利用readi来读信息到addr指定的位置
    ilock(f->ip);
    // r 是真正读取的字节数目
    if((r = readi(f->ip, 1, addr, f->off, n)) > 0)
      f->off += r;
    iunlock(f->ip);
  } else {
    panic("fileread");
  }

  return r;
}

// Write to file f.
// addr is a user virtual address.
int
filewrite(struct file *f, uint64 addr, int n)
{
  int r, ret = 0;

  if(f->writable == 0)
    return -1;

  if(f->type == FD_PIPE){
    ret = pipewrite(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].write)
      return -1;
    ret = devsw[f->major].write(1, addr, n);
  } else if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.
    // 这边除2确实是有一点奇怪，
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
    int i = 0;
    while(i < n){
      int n1 = n - i;
      if(n1 > max)
        n1 = max;

      begin_op();
      ilock(f->ip);
      if ((r = writei(f->ip, 1, addr + i, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op();

      if(r != n1){
        // error from writei
        break;
      }
      i += r;
    }
    ret = (i == n ? n : -1);
  } else {
    panic("filewrite");
  }

  return ret;
}
```
上面的filestat功能还是非常合理的，我们主要看下面两个函数。可能比较奇怪的是这边的`int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;`设置，其他地方看起来还是非常自然的。
### 8.14 system calls
有了上面这么多种类的文件操作之后，我们最后将其部署到system calls上。
```C
// Create the path new as a link to the same inode as old.
uint64
sys_link(void)
{
  char name[DIRSIZ], new[MAXPATH], old[MAXPATH];
  struct inode *dp, *ip;

  if(argstr(0, old, MAXPATH) < 0 || argstr(1, new, MAXPATH) < 0)
    return -1;

  begin_op();
  // 先找到 old 所对应的 ip，本质上是根据文件名，找到文件所对应的 inode
  if((ip = namei(old)) == 0){
    end_op();
    return -1;
  }

  ilock(ip);
  // 这个 inode 需要是一个文件，而不是文件夹
  if(ip->type == T_DIR){
    iunlockput(ip);
    end_op();
    return -1;
  }

  // 因为多了一个文件名来引用这个inode，所以需要增加 ip->nlink
  ip->nlink++;
  // 把这个信息更新到disk inode里去
  iupdate(ip);
  iunlock(ip);

  // 根据路径，找到对应的新文件所在的文件夹
  // 这必须得能找到
  if((dp = nameiparent(new, name)) == 0)
    goto bad;
  // 要对父亲文件夹做一点修改，所以需要ilock一下
  ilock(dp);
  // 如果父亲文件夹所用的设备和 ip->dev 是不同的，那这个inode并不能link上，因为设备是不一样的
  // 第二种情况便是，这个name已经有具体的文件名了，但是没有办法在dp上连接上，也就是父文件夹没法建立和这个新link建立的文件之间的关联
  if(dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0){
    iunlockput(dp);
    goto bad;
  }
  // 感觉是为了省事
  iunlockput(dp);
  // ip自然也得更新上
  iput(ip);

  // 范式
  end_op();

  return 0;

bad:
  // 失败了，先前加了，那么现在就减去 
  ilock(ip);
  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);
  end_op();
  return -1;
}


uint64
sys_unlink(void)
{
  struct inode *ip, *dp;
  struct dirent de;
  char name[DIRSIZ], path[MAXPATH];
  uint off;

  // 自然还是先找到路径位置
  if(argstr(0, path, MAXPATH) < 0)
    return -1;

  begin_op();
  // 先找爸爸文件夹
  if((dp = nameiparent(path, name)) == 0){
    end_op();
    return -1;
  }

  ilock(dp);

  // Cannot unlink "." or "..".
  // 这俩不能删掉的
  if(namecmp(name, ".") == 0 || namecmp(name, "..") == 0)
    goto bad;

  // 先找name是否存在，不存在那没办法unlink
  if((ip = dirlookup(dp, name, &off)) == 0)
    goto bad;
  ilock(ip);

  // 找到了底下的inode，但是它nlink小于1，说明要么错了，要么没有文件和这个inode进行连接
  if(ip->nlink < 1)
    panic("unlink: nlink < 1");
  // 是文件夹还能unlink取消，但是如果文件夹非空，对于unlink的要求太高了，因为要遍历，这并不好
  if(ip->type == T_DIR && !isdirempty(ip)){
    iunlockput(ip);
    goto bad;
  }

  memset(&de, 0, sizeof(de));
  // 利用先前找到的off，以及父亲文件夹用的dp，把0作为信息写到dp的data block部位，表示从文件夹索引的directory entry删掉了
  // 这要是还失败，那这辈子有了
  if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
    panic("unlink: writei");
  // 如果是这样，就不太需要考虑dp了，可以考虑ip的情况
  if(ip->type == T_DIR){
    dp->nlink--;
    iupdate(dp);
  }
  iunlockput(dp);

  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);

  end_op();

  return 0;

bad:
  iunlockput(dp);
  end_op();
  return -1;
}
```

### 结语
到这里，我们对于filesystem的解读工作可以算是完成了。