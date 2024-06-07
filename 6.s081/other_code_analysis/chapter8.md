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
========    next               next
|      | ----------> ====== ----------> ======
| head |             | b0 |             | b1 | .... 
|      | <---------- ====== <---------- ======
========    prev               prev
```
研究一下基本单元 buf 的信息：
```C
struct buf {
  int valid;   // has data been read from disk? 其实就是说这个 buf 是否本身是对 disk 内信息的一个备份
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
  if(!holdingsleep(&b->lock))
    panic("brelse");
  // b->lock 被 holding 时表示该 buf 正在被使用
  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    // 遭到 release 的块，将会成为 head 所指向的下一个块
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

### 8.7 Block Allocator
感觉单拎出来东西再去看文件系统的代码有点奇怪，不如我们先把文件系统相关的东西全部看完，再开始看代码。

这边提到 mkfs 的作用，我们在完成了这章的阅读后在记得回溯一下这个工具。

先尝试分析 balloc 
```C
// Allocate a zeroed disk block.
static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;

  bp = 0;
  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}
```