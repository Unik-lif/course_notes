## Chapter 8
这一章可能会相当之长，做好心里预期。
### 8.3 buffer cache
首先是对 buffer 的初始化，这边光看代码可能看不出什么东西，画个图就很清晰了，在为每个 b->lock 进行初始化的同时，我们大概得到的 buffer 效果如下图所示：
```C
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```
效果图：
```
bcache整体：

========    next               next
|      | ----------> ====== ----------> ======
| head |             | b0 |             | b1 | .... 
|      | <---------- ====== <---------- ======
========    prev               prev
```
研究一下基本单元 buf 的信息：
```C
struct buf {
  int valid;   // has data been read from disk? 其实就是说这个 buf 是否在 disk 中有同样的信息，也就是内存和缓存是否同步
  int disk;    // does disk "own" buf? 其实就是说 buf 是否已经把信息传送回给 disk
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
};
```
之后我们再考虑一下研究读写相关的事情：
```C
// Return a locked buf with the contents of the indicated block.
// 此处的 blockno 似乎就是 sector number
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;
  // 调用 bget 来进行读
  b = bget(dev, blockno);
  // 如果 b->valid 为 0 ，则说明当前的这个 buf 并不是对于 disk 内容的拷贝
  if(!b->valid) {
    // 那没有办法，必须得去读这个 virtio_disk 了
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}

// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  // 当前进程是否持有这个b的锁？
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  // 要把 b 的内容拿去写回到 virtio_disk 之中
  virtio_disk_rw(b, 1);
}

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;
  // 对 bcache 的访问需要加上锁
  acquire(&bcache.lock);

  // Is the block already cached?
  // 对于 bcache 中的 buf 单元进行遍历，所以要用 bcache.lock 加锁
  // 从 bcache.head 顺序访问，
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    // 设备号和 blockno 值都相同，表示确实对该 buf 单元达成了引用
    // 并且似乎确实已经做了 cached ，可以直接返回我们的 buf 单元
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      // 跑到这边的时候，我们对于 bcache 的遍历已经结束了，因此已经可以进行遍历了
      release(&bcache.lock);
      // 使用时还得把这边的 buf 的这个锁给拿到， bget 会返回这个 buf 块
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  // 该单元，即 dev 对应的 blockno 尚没有构成引用
  // 倒序遍历，这样能够更快找到我们希望置换掉的东西
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    // 对于没有使用过的单元，覆盖写他的值，返回这个新创建的 buf 单元
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      // 为什么 bcache.lock 要在对于 b 进行了操作之后再释放？
      // 因为 bcache 的真实目的其实是为了保证最多每个 sector （由 blockno 对应）只有一个 cached buffer
      // 把检查和操作放在一起，作为一个完整的原子操作
      release(&bcache.lock);
      // b->refcnt >= 1 并且 b->lock 被加了锁，说明该 buf 单元处于忙碌的状态
      // b->lock 并没有特别大的同步需求，其主要还是为了实现对 buf 内容的保存
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}

// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  // 检查一开始acquiresleep的是否就是这个进程
  if(!holdingsleep(&b->lock))
    panic("brelse");
  // b->lock 被 holding 时表示该 buf 正在被使用
  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    // 遭到 release 的块，将会成为 head 所指向的下一个块
    // 从原来的链表中直接截取出来，然后放到队列的最前面
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```
到这边，为了 Lab 8 所需要的代码分析任务算是完成了。

### 8.6 Logging Layer
首先看向begin_op函数：
```C
struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
struct log log;

// called at the start of each FS system call.
void
begin_op(void)
{
  // 首先获得记录日志的锁
  acquire(&log.lock);
  while(1){
    // 如果还在commiting状态，可能还得等一会儿
    if(log.committing){
      // 释放掉log.lock锁，并根据log来作为信号量来睡觉，等待被唤醒
      sleep(&log, &log.lock);
      // 超出了当前能用的资源空间
      // log.lh.n：现有的空间
      // log.outstanding + 1 如果要再加上一个文件系统调用
      // MAXOPBLOCKS 每一个文件系统调用所需的最大的空间
      // 相对保守的分配策略
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      // outstanding表示现在有多少个文件系统调用正在被调用
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```
保证现在的logging系统并没有在commiting，且现在有足够的log space用来放置当前调用的写操作。

下面分析log_write：
```C
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    // 已经有了就把它放在同一个操作，即absorbation，以节省资源
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    // 增加一个ref，防止被驱逐回去
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```
主要是把b所对应的blockno记录到log之中，并且通过absorbtion，来减少一些资源的消耗。

下面分析end_op：
```C
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  // 表示某个op要结束了，所以先减少一下outstanding
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    // 当所有的outstanding都结束了，是时候该commit了
    do_commit = 1;
    log.committing = 1;
  } else {
    // 否则，还有其他一些行为没有结束
    // 但是log.committing是0，原来在begin_op中卡住的行为，现在需要被唤醒了
    // 一个是log.outstanding导致的空间不够，一个是因为先前log.committing为1时卡住的，一共两种情况
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.

    // 感觉释放也挺有道理的，毕竟在log上的更新已经完成了
    // 很识趣的是，其实一个变量，也就是committing已经限制了其他的begin_op没法在committing的时候影响我们的commit操作
    // 所以这边释放是完全可行的
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```
下面需要对commit做更加详细的研究：
```C
static void
commit()
{
  // 均建立在log.lh.n大于0的前提上，否则说明log中其实没有存什么东西，没有必要做更新
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit，真正把header写到disk的步骤，过了这一步后，数据都能恢复了
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log，用0写，清楚掉无效数据
  }
}

// Copy modified blocks from cache to log.
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    // 从log.lh.block[tail]这个cache位置，写到log.start+tail+1的位置
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to);  // write the log
    // 写完了自然释放掉
    brelse(from);
    brelse(to);
  }
}

// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
static void
write_head(void)
{
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

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    if(recovering == 0)
      bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}
```
继续看recover_from_log函数：
```C
static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```
看起来很简单，如果log的header已经弄好了，可以直接从disk读log block到内存中，用来恢复。之后通过install_trans真正地重新做一遍log里头要求我们做的事情。

最后write_head写一个空的，用来清除log。

虽然细节没有彻底搞明白，但是看起来还是挺靠谱的。


## 文件系统代码梳理
对于文件系统还是感觉非常奇怪，我们还是多花一点时间读一下代码的情况吧，看看一步一步都到底是怎么做的。

首先，文件系统是其他工具制造出来的，我们需要看一下 mkfs 程序。

在 Makefile 中有这个程序的生成和使用过程：
```
fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```
下面是该工具所对应的代码：
```C
int
main(int argc, char *argv[])
{
  int i, cc, fd;
  uint rootino, inum, off;
  struct dirent de;
  char buf[BSIZE];
  struct dinode din;

  // 判断当前 qemu 环境中的 int 大小是否是 4 个字节
  static_assert(sizeof(int) == 4, "Integers must be 4 bytes!");

  // 东西要放到 fs.img 之中
  if(argc < 2){
    fprintf(stderr, "Usage: mkfs fs.img files...\n");
    exit(1);
  }
  // 判断 block size 是否是 dinode 和 dirent 大小的倍数
  // BSIZE 大小为 1024 字节
  /*
    dinode 表示 data node，刚好 64 字节
    struct dinode {
      short type;           // File type
      short major;          // Major device number (T_DEVICE only)
      short minor;          // Minor device number (T_DEVICE only)
      short nlink;          // Number of links to inode in file system
      uint size;            // Size of file (bytes)
      uint addrs[NDIRECT+1];   // Data block addresses
    };
    后面的 dirent 正好是 16 字节
  */

  assert((BSIZE % sizeof(struct dinode)) == 0);
  assert((BSIZE % sizeof(struct dirent)) == 0);
  // 打开 fs.img 这个文件，准备对其进行编辑
  fsfd = open(argv[1], O_RDWR|O_CREAT|O_TRUNC, 0666);
  if(fsfd < 0){
    perror(argv[1]);
    exit(1);
  }
  // Disk layout:
  // [ boot block | sb block | log | inode blocks | free bit map | data blocks ]
  
  // 需要注意，Inode 的单位要小于 Block
  // 1 fs block = 1 disk sector
  // nmeta 表示 metadata 所对应的 metadata 所占据的 blocks 一共有多少个
  // 首先是空的 block 以及 superblock ，之后记录了 nlog 和 ninodeblocks 以及 nbitmap 的总数
  // nlog: 存放 log 信息的 block 总数，一共是 30 个
  // ninodeblocks: 存放 inode 所占据的块总数，通过 NINODES / IPB + 1 来计算，一共有 200 个 Inodes ，每个 block 有 IPB 个 Inode ，然后再 +1 表示富余
  // nbitmap: 用每个 bit 来表示当前的空间是否得到了分配， FSSIZE/(BSIZE*8) + 1 中 FSSIZE ， FSSIZE 表示文件系统的总 blocks 数目，由于我们给 nbitmap 赋予的职能是利用 bit 来表示 block 的占用情况，因此每个 block 恰好可以表示 1024 * 8 这么多个 block 的使用情况，因此所占据的用于看 block 分配情况的 nbitmap block 总数可以通过上面的方式计算得到

  // nmeta 表示存放 metadata 的数据，这些数据是用来管理的，和作为 data block 的 nblocks 需要区分开来
  nmeta = 2 + nlog + ninodeblocks + nbitmap;
  // 剩下的 block 全部用于存放数据
  nblocks = FSSIZE - nmeta;

  sb.magic = FSMAGIC;
  // 这边是为了解决一些大小端序的问题
  // 不过我的疑问在于魔数这边，凭什么他就不需要转换？不也跟其他变量一样被xv6用了吗？
  sb.size = xint(FSSIZE);
  sb.nblocks = xint(nblocks);
  sb.ninodes = xint(NINODES);
  sb.nlog = xint(nlog);
  sb.logstart = xint(2);
  // 参考上面的 disk layout
  sb.inodestart = xint(2+nlog);
  sb.bmapstart = xint(2+nlog+ninodeblocks);
  
  printf("nmeta %d (boot, super, log blocks %u inode blocks %u, bitmap blocks %u) blocks %d total %d\n",
         nmeta, nlog, ninodeblocks, nbitmap, nblocks, FSSIZE);

  freeblock = nmeta;     // the first free block that we can allocate，前面的那些 blocks 早就已经被占用干净了

  // wsect 函数把 buf 内的数据写到 i sector 所对应偏移量位置
  // 这边看起来其实就是全部初始化设置为 0 一次
  // 每个 sector 对应 1024 字节
  // 其实就是不停往下写了这么多东西，通过 lseek 让 fsfd 向下移动
  for(i = 0; i < FSSIZE; i++)
    wsect(i, zeroes); // write sect 和 write inode 不是同一个粒度

  memset(buf, 0, sizeof(buf));
  // 将 sb 的信息写入到 buf 之中，但现在的 sb 还没有真正写到 sb 所对应的 filesystem 块位置
  memmove(buf, &sb, sizeof(sb));
  // 将 sb 信息通过 buf 真正写到 filesystem 的 sb 理应存在的位置
  wsect(1, buf);

  // 仔细研究一下 ialloc 函数
  // 刚刚做了初始化，反正空间全部给定了，现在准备写东西
  rootino = ialloc(T_DIR);
  assert(rootino == ROOTINO);
  // 刚刚分配了 ROOTINO ，它所对应的 data inode 我们确实已经分配了
  bzero(&de, sizeof(de));
  de.inum = xshort(rootino);
  strcpy(de.name, ".");
  // inode append derent to the rootino
  iappend(rootino, &de, sizeof(de));

  bzero(&de, sizeof(de));
  de.inum = xshort(rootino);
  strcpy(de.name, "..");
  iappend(rootino, &de, sizeof(de));
```

```C
// 文件系统尚没有构成，可以一以相对便捷的方式来做
uint
ialloc(ushort type)
{
  uint inum = freeinode++;
  // 64 bytes
  struct dinode din;
  
  bzero(&din, sizeof(din));
  din.type = xshort(type);
  din.nlink = xshort(1);
  din.size = xint(0);
  // write inode
  winode(inum, &din);
  return inum;
}

void
winode(uint inum, struct dinode *ip)
{
  char buf[BSIZE];
  uint bn;
  struct dinode *dip;
  // 对应的是从 nlog 之后的 inode block 开始计量，在 inode block 中存放文件目录等信息，
  // 会对上一些基本的索引情况，包括直接索引和间接索引
  bn = IBLOCK(inum, sb);
  // 得到 inum 所对应的 block number
  // read sector 把东西读到 buf 中
  // 大小为 BSIZE
  // inode number => block number => read block to the buf => buf
  rsect(bn, buf);
  // 先通过 rsect 得到 buf 信息，然后对这个 buf 信息中我们需要修改的特定的 inum 做了修改之后
  // 重新把 buf 写回这个 block
  // 其实还是非常合理的
  // 先找到 block ，然后读 block 信息，根据 inum 找到当前 block 中的需要修改的 inode 位置，修改 block 信息对应的 buf
  // 最后再把 buf 写回到 block 之中
  dip = ((struct dinode*)buf) + (inum % IPB);
  // 把 *ip 的信息赋值给 *dip
  *dip = *ip;
  wsect(bn, buf);
}

void
rsect(uint sec, void *buf)
{
  // 送到对应的 sec * BSIZE 偏移量位置处
  if(lseek(fsfd, sec * BSIZE, 0) != sec * BSIZE){
    perror("lseek");
    exit(1);
  }
  // read fsfd to buf
  if(read(fsfd, buf, BSIZE) != BSIZE){
    perror("read");
    exit(1);
  }
}

void
wsect(uint sec, void *buf)
{
  if(lseek(fsfd, sec * BSIZE, 0) != sec * BSIZE){
    perror("lseek");
    exit(1);
  }
  if(write(fsfd, buf, BSIZE) != BSIZE){
    perror("write");
    exit(1);
  }
}

// iappend, inode append.
void
iappend(uint inum, void *xp, int n)
{
  char *p = (char*)xp;
  uint fbn, off, n1;
  struct dinode din;
  char buf[BSIZE];
  uint indirect[NINDIRECT];
  uint x;

  rinode(inum, &din);
  off = xint(din.size);
  // printf("append inum %d at off %d sz %d\n", inum, off, n);
  while(n > 0){
    fbn = off / BSIZE;
    assert(fbn < MAXFILE);
    if(fbn < NDIRECT){
      if(xint(din.addrs[fbn]) == 0){
        din.addrs[fbn] = xint(freeblock++);
      }
      x = xint(din.addrs[fbn]);
    } else {
      if(xint(din.addrs[NDIRECT]) == 0){
        din.addrs[NDIRECT] = xint(freeblock++);
      }
      rsect(xint(din.addrs[NDIRECT]), (char*)indirect);
      if(indirect[fbn - NDIRECT] == 0){
        indirect[fbn - NDIRECT] = xint(freeblock++);
        wsect(xint(din.addrs[NDIRECT]), (char*)indirect);
      }
      x = xint(indirect[fbn-NDIRECT]);
    }
    n1 = min(n, (fbn + 1) * BSIZE - off);
    rsect(x, buf);
    bcopy(p, buf + off - (fbn * BSIZE), n1);
    wsect(x, buf);
    n -= n1;
    off += n1;
    p += n1;
  }
  din.size = xint(off);
  winode(inum, &din);
}
```
filesystem 是在 forkret 中做的初始化。而 forkret 这个函数也是只在刚进来的时候执行一次。
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
  readsb(dev, &sb);
  if(sb.magic != FSMAGIC)
    panic("invalid file system");
  initlog(dev, &sb);
}

// Read the super block.
static void
readsb(int dev, struct superblock *sb)
{
  struct buf *bp;

  bp = bread(dev, 1);
  memmove(sb, bp->data, sizeof(*sb));
  brelse(bp);
}
```
在这边通过 fsinit 来初始化，然后用 readsb 来读取信息到 sb 之中。特别的，由于 superblock data 信息是存在 device 中的第一个块中，可以轻松这么读出来。 bread 读完之后自己释放就了得了。

马上解析该数据的魔数是否是对的。

不过这边最巧妙的其实是 first 这个变量的使用，它被定义为 static int 类型，也就是能够在全局中被使用，但是只能被初始化一次，这个良好的特性使得我们其他进程访问到的 first 和第一个进程访问到的 first 是同一个，也因此能够避免我们多次重复初始化文件系统。

对应的链接是这个，还挺新奇的，不过也有可能是我的基础太烂了，呜呜哭了。https://stackoverflow.com/questions/5567529/what-makes-a-static-variable-initialize-only-once

那这边这个文件系统的初始化还怪快的？

接下来的重头戏是 initlog ，用来让文件系统能够有持久化的特性（崩溃一致性）
```C
void
initlog(int dev, struct superblock *sb)
{
  if (sizeof(struct logheader) >= BSIZE)
    panic("initlog: too big logheader");

  initlock(&log.lock, "log");
  // 这边的设置信息疑似是在 mkfs 的时候写的
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
```