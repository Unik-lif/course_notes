## 实验概览
本质上要求也很简单，但是估计做起来可能会比较麻烦。具体来说要做一个 cow fork。这需要我们把父进程和子进程的 PTE 均写成 not writable。

当其中一个进程尝试写这些页时，先在物理内存中分配一个这样的页，把原本的 page 内容拷贝到这个新的页之中。对于新的页，设置其为 writable ，再做对应的修改。

在 reasonable plan of attack 之中有很详细的实验说明指导，其中第三点感觉非常值得注意。这意味着我们需要维护一下 refcount 这个数据结构。

需要标志一下某个页是否处于 COW 映射状态下，注意到 PTE 的 8-bit 与 9-bit 是 RSW 位置，可以用来做标记。本来其实最理想情况下是 RSW 足够多，我们可以直接把 refcount 写到这边，不过看起来没辙了。

注意， COW 的本质是对于已经 COW-mapped 的页在异常发生之后进行操作。

似乎已经足够可以开始写我们的代码了。

## 实验步骤
我们依照实现的建议顺序来做，先尝试通过 cowtest 。

一开始这个页呢还是正常的，直到我们检测到了 uvmcopy 的运行，这时候我们才要对原本的页的性质做一些修改。需要加上 PTE_COW 标志，并且去掉 PTE_W 标志

copyout 为什么需要做修改？
- 因为 User pagetable 是有可能不完整的，在 copyout 发生我的意思是他有可能还是处于 COW 状态，也就是并没有分配所谓的物理地址空间，这边的拷贝可能原则上是不成立的？

freeproc 也需要做一些修改，否则就直接把 cow ，但是还没有自行复制一份的内存页全部给释放了。为此，我们需要修改一下 kinit 这边的系统。

异常：
不知道为什么父亲进程也给我弄成 zombie 了，应该需要想办法解决这件事情。

卡机原因：写代码的时候要小心，不要把 trampoline 一并给 free 掉了。
```
 .. ..511: pte 0x0000000021fd5001 pa 0x0000000087f54000                                                                                                                                                           .. .. ..510: pte 0x0000000021fd5cc7 pa 0x0000000087f57000                                                                                                                                                        .. .. ..511: pte 0x0000000020001c4b pa 0x0000000080007000                                                                                                                                                       freed memory: 0x0000000087f50000                                                                                                                                                                                 freed memory: 0x0000000087f51000                                                                                                                                                                                 freed memory: 0x0000000087f4f000                                                                                                                                                                                 freed memory: 0x0000000087f4e000                                                                                                                                                                                 freed memory: 0x0000000087f4d000                                                                                                                                                                                 freed memory: 0x0000000087f4c000                                                                                                                                                                                 freed memory: 0x0000000087f4b000                                                                                                                                                                                 freed memory: 0x0000000087f4a000                                                                                                                                                                                 freed memory: 0x0000000087f49000                                                                                                                                                                                 freed memory: 0x0000000087f48000                                                                                                                                                                                 freed memory: 0x0000000087f47000                                                                                                                                                                                 freed memory: 0x0000000087f46000
freed memory: 0x0000000087f45000
freed memory: 0x0000000087f44000
freed memory: 0x0000000087f43000
freed memory: 0x0000000087f42000
freed memory: 0x0000000087f41000
freed memory: 0x0000000087f40000
hi
p->pid: 3 p->name: echo
freed memory: 0x0000000087f57000
```
我们在函数 freeproc 中找到了可能的原因：
```C
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  printf("p->pid: %d p->name: %s p->trapframe: %p\n", p->pid, p->name, p->trapframe);
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```