## 实验概览
在多核情况下重新设计数据结构以避免忙等。

> The basic idea is to maintain a free list per CPU, each list with its own lock. 
- 需要为每一个 CPU 都维持一个 free list , 每个 list 都有他们用于操作的自己的 lock 结构
> Allocations and frees on different CPUs can run in parallel, because each CPU will operate on a different list. 
- 如果把上面这件事做了这件事情应该不麻烦
> The main challenge will be to deal with the case in which one CPU's free list is empty, but another CPU's list has free memory; in that case, the one CPU must "steal" part of the other CPU's free list. Stealing may introduce lock contention, but that will hopefully be infrequent.
- 特殊情况，一个 CPU 的 free list 是空的，即当前该 CPU 视角下内存已经被用尽。但是另外一个 CPU 则是有空闲内存。需要让前面的 CPU 能够盗取另外一个 CPU 的空闲队列用来使用。


### 实验记录：

#### Memory allocator

> 卡在了一个莫名其妙的地方：我单知道要避免找自己了，但是居然忘记 for 的语义了，是不是代码写的太少了？

对于这个，我的理解应该是没有什么问题的，主要还是因为卡在莫名其妙的地方。严格记录一下，我在低级错误大全之中加入了一个模块，引以为鉴。

#### buffer cache

感觉这个实验有点难，我们先尝试把脉络梳理一下：

> you'll see that bcache.lock protects the list of cached block buffers, the reference count (b->refcnt) in each block buffer, and the identities of the cached blocks (b->dev and b->blockno).

主要的问题在于， bcache.lock 的粒度似乎还是太大了，从而让很多行为操作都依赖他。为此可能要盲等很久？

> We suggest you look up block numbers in the cache with a hash table that has a lock per hash bucket.

看起来我们需要彻底革除原来的链表结构，转用 hashtable来做。我们依旧会保留原本的数组结构以方便访问，但是我们可能会用哈希表的方式来对其进行组织，在此之前，我们再仔细研究一下原本的链表结构，感觉我对它的理解还做不到鞭辟入里。

> eviction

这里的 eviction 是什么意思？其实就是 block 数目要远远多于我们的 buf 数目。特别的，在我们这边进行 eviction 之前，我们要先 release 完。

可能性：

我们依照设计的哈希表结构尝试搭建我们的流程，但是似乎在 bcachetest 中遇到了这样的问题。

```
bget findcache:
scause 0x000000000000000d
sepc=0x000000008000121a stval=0x0000000000000000
panic: kerneltrap
```
追溯了一下，问题发生源于 dev = 1 即输出时的情况，而且是在我们疑似能够在 findcache 函数中找到的分支。

### 方案设计
我们重新尝试设计一下方案，尽可能清楚地描述我们的想法，以此避免一些非常细微的问题。

首先，我们会把 buf 按照 hashTable 进行编排，我们的大体思路是，通过计算 b->dev 和 b->blockno 的和的哈希值，确认选取 hashTable 的哪一个 bucket 来进行添加。

因此，我们在初始化时的思路还是相对清楚的：先把所有的 buf 块加入到 hashtable 之中，之后再按照 bget 的方式来进行选用。在初始化的时候，不要在 dev 和 blockno 中加入额外的值。
```C
// Set up a hash table for the bcache.
// Use hash table instead of the list.
void
binit(void)
{
  char name[NAME_LEN];
  struct buf *next_entry;
  initlock(&bcache.lock, "bcache");
  for(int i = 0; i < HASH_BUCKET; i++){
    snprintf(name, NAME_LEN, "bcache.hashbuck%d", i);
    initsleeplock(&hashtable.lock[i], name);
    hashtable.bucket[i].next = 0;
    memset(name, 0, NAME_LEN);
  }
  // Create linked list of buffers
  for(int i = 0; i < NBUF; i++){
    snprintf(name, NAME_LEN, "buffer%d", i);
    initsleeplock(&bcache.buf[i].lock, "buffer");
    if(hashtable.bucket[i % HASH_BUCKET].next != 0){
      next_entry = hashtable.bucket[i % HASH_BUCKET].next;
      next_entry->prev = &bcache.buf[i];
      bcache.buf[i].next = next_entry;
    }
    hashtable.bucket[i % HASH_BUCKET].next = &bcache.buf[i];
    bcache.buf[i].prev = &hashtable.bucket[i % HASH_BUCKET];
    bcache.buf[i].num = i;
    bcache.buf[i].timestamp = ticks;
    
    memset(name, 0, NAME_LEN);
  }
  printhash();
}
```

在完成了初始化后，我们来分析 brelse 可能遇到的情况。

brelse 的工作应该是相对简单的，在我们删掉了原本的链表结构后，需要做的操作可能也就是当 refcnt == 0 时，对时间参数进行更新：
```C
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  // printf("brelse\n");
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // printf("brelse b->refcnt: %d b->num: %d\n", b->refcnt, b->num);
    // no one is waiting for it.
    
    b->timestamp = ticks;    
  }
  
  release(&bcache.lock);
  
  // printf("ahh?\n");
}
```
需要注意，我们并不需要设置 b->valid = 0 ，也不需要针对 dev 和 blockno 等相关信息做一些较大的修改，即便 brelse 操作了这个块，只是说明短期我们不再使用它，并不代表它所代表的信息就可能是过时了的，既然不会过时，我们就没有必要对 valid 的值做相关的修改。因为下一次尝试访问同样的 dev 和 blockno ，肯定是要先去查看缓存中是否有这个块的。

接下来是我们的重头戏，也就是 bget 的实现，经过简单的梳理，我们将情况分成下面的三种类型：

> 能够直接在提供 dev 以及 blockno 的情况下，找到对应 hashtable bucket 中我们想要的块结构
```C
// Find whether the given dev and blockno
// Situates in the hashtable
// If not, return 0.
static struct buf*
findcache(uint dev, uint blockno){
  struct buf* b;
  uint hash_num = (dev + blockno) % HASH_BUCKET;
  acquire(&hashtable.lock[hash_num]);
  // go through the hashtable bucket till we reach the end
  // don't start our traverse from the sentinel.
  for(b = hashtable.bucket[hash_num].next; b != 0; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      b->timestamp = ticks;
      release(&hashtable.lock[hash_num]);
      return b;
    }
  }
  release(&hashtable.lock[hash_num]);
  return 0;
}
```
需要注意，没有必要在条件判断中加上 b->refcnt == 0 ，原因在上面已经说了，哪怕 brelse 操作了这个块，也不代表我们不能使用它。

> 能够直接在提供 dev 以及 blockno 的情况下，找到对应 hashtable bucket 中的空闲块，虽然不是我们想要的块结构（比如 blockno 和 dev 的值其实并不吻合）

其实这里似乎也有两种情况，第一种情况是，我们所富余的那个块并不是因为它先前的 hash 后的结果被存放到这个 bucket 之中的，而是一开始初始化时被迫找个方式来放它。第二种情况，我们富余的那个块是之前因为 hash 被放到这个 bucket 中的。但是这些都不是很大的问题，两种情况应该是以同样的方式来处理的。

我们先在特定 bucket 中进行遍历，找到 timestamp 最小的那个 bcache 所对应的号，存放在 evict_buf_num 中，然后对其进行操作。我们需要更新 dev 和 blockno 值，并且由于这个块并不是之前就存在的，需要设置 valid 为 0 ，以便后续我们对值进行更新。
```C
  // printf("bucketevict function:\n");
  int evict_num = NBUF;
  int evict_buf_num = 0;
  uint time = 0xffffffff;
  uint hash_num = (dev + blockno) % HASH_BUCKET;
  struct buf* next;
  struct buf* next_hash;
  acquire(&hashtable.lock[hash_num]);
  struct buf* b = hashtable.bucket[hash_num].next;
  for(int i = 0; b != 0; b = b->next, i++){
    if(b->refcnt == 0 && b->timestamp < time){
      time = b->timestamp;
      evict_num = i;
      evict_buf_num = b->num;
    }
  }
  // Succeed to find the buf we want to evict. 
  if(evict_num != NBUF){
    // printf("find evict case 1\n");
    b = &bcache.buf[evict_buf_num];
    // access the evict_num th entry
    b->dev = dev;
    b->blockno = blockno;
    b->valid = 0;
    b->refcnt = 1;
    b->timestamp = ticks;
    // printf("bucket_eviction b->refcnt: %d b->num: %d\n", b->refcnt, b->num);
    release(&hashtable.lock[hash_num]);
    return b;
  }
```

想来想去感觉似乎没有什么明显的问题，感觉有一定概率是我迭代的脚步太快了？或者实际上链表实现上有一些问题，被我忽略了。

上面的代码可能没啥用了，估计只是一个用来帮助理清思路的事情。

或许我们可以考虑慢慢迭代，重新回到初版，然后再一点点加上功能跑跑看？

我回到了初版，用半个小时快速重新写了一版，到这里其实我们大体上的思路已经是非常明确的了，实现起来也不是特别费劲。

但是有一个问题需要注意：我看到应该没有人在解析里头似乎提到**下面这一点**，不过自己运气不错还是慢慢把这个问题解决了。

> 当我们出现了 PAGE FAULT 的时候，往往并不是 valid 或者相关同步的事情出现了问题，往往其实是我们在用链表做链接的时候出现了一些低级错误，比如对于 0 指针的非法操作等等。

需要自己仔细排查相关的问题，其中一个比较合理的方式是，尽可能构造容易出现需要我们涉及页表调用的情况，比如一开始把所有的 block 都放在第一个 hashtable entry 之中，这样我们很快就会出现需要从第一个 hashtable entry 偷窃 buf 单元到其他 hashtable entry 的情况。

然后还有一些零碎的问题：
- 其实我们可以不考虑用 ticks 来做，本质上我们只要找到一个 refcnt 为 0 的单元，就可以做替换了，这样不会影响最后的判分，但是可能会影响效率和速度。不过很好的一点是，当我们事先这么干后，向下迭代会变得非常容易。增量式迭代看起来是一个非常好的编程实践，该方法论也帮助我写清楚了 cow fork 实验。
- manywrites 可能会出现死锁的情况，大概率是有多个进程在 evict 的情况（其实我个人觉得用 evict 来形容并不是很贴切，倒不如说是在当前 hashtable entry 中没有找到空闲的 buf 块，然后转而向其他 hashtable entry 寻找的情况），A 进程的遍历和 B 进程的遍历对于他们当前所持锁之间出现了争用，进而触发死锁。解决这个问题其实也挺简单，我们让他们在遍历的时候不要尝试获得第二个锁，让每个进程在 bget 中最多一口气获得一个锁，就不会产生争用了。当然，这样会对代码有一个要求，也就是 critical section 要分割的比较清楚，否则就会出现同步上的问题。
- 其实有不少在编程时出现的匪夷所思的问题可能是我们在使用 hashtable 数据结构时在数据结构代码实现上出现了误差，所以一定要小心。避免这个问题也挺简单，写一个遍历 hashtable 的 debug 函数就好了，然后我们可以检查到底会不会出现 buf 数量突然就和 NBUF 不同的情况，这样就能判断链表是不是写错了。

最后是判分，如下所示，感觉收获真的挺大的。
```
== Test running kalloctest == 
$ make qemu-gdb
(79.8s) 
== Test   kalloctest: test1 == 
  kalloctest: test1: OK 
== Test   kalloctest: test2 == 
  kalloctest: test2: OK 
== Test kalloctest: sbrkmuch == 
$ make qemu-gdb
kalloctest: sbrkmuch: OK (7.9s) 
== Test running bcachetest == 
$ make qemu-gdb
(6.3s) 
== Test   bcachetest: test0 == 
  bcachetest: test0: OK 
== Test   bcachetest: test1 == 
  bcachetest: test1: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (109.4s) 
== Test time == 
time: OK 
Score: 70/70
```