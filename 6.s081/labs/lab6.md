## 实验概览
本质上要求也很简单，但是估计做起来可能会比较麻烦。具体来说要做一个 cow fork。这需要我们把父进程和子进程的 PTE 均写成 not writable。

当其中一个进程尝试写这些页时，先在物理内存中分配一个这样的页，把原本的 page 内容拷贝到这个新的页之中。对于新的页，设置其为 writable ，再做对应的修改。

在 reasonable plan of attack 之中有很详细的实验说明指导，其中第三点感觉非常值得注意。这意味着我们需要维护一下 refcount 这个数据结构。

需要标志一下某个页是否处于 COW 映射状态下，注意到 PTE 的 8-bit 与 9-bit 是 RSW 位置，可以用来做标记。本来其实最理想情况下是 RSW 足够多，我们可以直接把 refcount 写到这边，不过看起来没辙了。

注意， COW 的本质是对于已经 COW-mapped 的页在异常发生之后进行操作。

似乎已经足够可以开始写我们的代码了。

## 实验步骤
我们依照实现的建议顺序来做，先尝试通过 cowtest 。我们不讲实验的细节，关键讲一下自己在写实验时，遇到的一些（很多）坑点。

一开始这个页呢还是正常的，直到我们检测到了 uvmcopy 的运行，这时候我们才要对原本的页的性质做一些修改。需要加上 PTE_COW 标志，并且去掉 PTE_W 标志

copyout 为什么需要做修改？
- 因为 User pagetable 是有可能不完整的，在 copyout 发生我的意思是他有可能还是处于 COW 状态，也就是并没有分配所谓的物理地址空间，这边的拷贝可能原则上是不成立的？

freeproc 也需要做一些修改，否则就直接把 cow ，但是还没有自行复制一份的内存页全部给释放了。为此，我们需要修改一下 kinit 这边的系统。

## 异常：
### 不知道为什么父亲进程也给我弄成 zombie 了，应该需要想办法解决这件事情。
我们想要达成的小目标是，至少跑 `echo hi` 不会卡死，然而似乎并不容易。

仔细打表发现最后卡在了莫名其妙的地方， gdb 上卡住的指令似乎是 `addi sp, sp(0)` ，对应 `010101` 这样的内存信息，经过检查发现是没有把 `kfree` 改好，一定要把 `memset` 放到内存释放里头来做，不然会覆写掉我们的指令位置，因为有一部分 `kfree` 并不会真的 `free` 掉内存。

### cowtest
```
$ cowtest
simple: ok
simple: ok
three: ok
three: ok
three:
```
到实验结点这个位置，我们其实还没有对 `copyout` 做修改，我们先修改一下对应的测试脚本 `cowtest`。对应 `gdb` 是卡在这个位置：
```
0x0000003ffffff000 in ?? ()
=> 0x0000003ffffff000:  14051573                csrrw   a0,sscratch,a0
```
发现是内存分配完了，`killed`就好，于是过了 `three`，卡在了后面的 `file`：需要修改 `copyout`.
```
$ cowtest
simple: ok
simple: ok
three: ok
three: ok
three: ok
file: error: read failed
error: read failed
error: read failed
```
修改 `copyout` 时要注意，首先我们需要拷贝完原来的物理内存信息，然后再做后续的修改处理：
```C
  if(flags & PTE_COW && !(flags & PTE_W)){
    mem = kalloc();
    if(mem){
      uvmunmap(pagetable, va0, 1, 0);
      memmove(mem, pa0, PGSIZE);
      memmove((void *)(mem + (dstva - va0)), src, n);
      mappages(pagetable, va0, PGSIZE, mem, flags | PTE_W);
    }
  } else {
    memmove((void *)(pa0 + (dstva - va0)), src, n);
  }
```
到这一步 `cowtest` 就完成了。
### Usertests
- 卡在了 `test copyout` ，可能是有一些边角料的情况？
挺好处理地，加上一些判定条件就好了。

- 卡在了 `test sbrkbugs`
遇到的问题似乎是这个：不知道 `sbrk(-0x10512)` 触发了什么错误，看起来像是不小心把某个部分清空了？
```
flags: 0x0000000000000000 0x0000000000000000
usertrap(): unexpected scause 0x000000000000000c pid=6
            sepc=0x0000000000005674 stval=0x0000000000005674
```
找了老半天原因原来是卡在这边，还怪好玩的。。。其实就是通过 `walk` 得到的地址是 `0`，这个异常没有好好处理，所以就卡在这里了。

- 卡在了 `test reparent`
居然对总空闲内存有一定的要求，要求前后的 `free page` 数目是一致的，这个要求其实还挺高的？

卡在了这个部分，我们学一下 Lec 12。

## 指导：
罗伯特莫里斯的实验技巧：
- 增量式代码撰写，一点点地测试与 debug：如果我们先完全完成我们的项目时，我们很有可能还没有对这个项目的细节有充分的了解，除非我们已经有了一些`debug`时的经验。
- 决定下一步：通过人为构造 `panic` 来决定下一步的需求。
- 面向测试来学习
- 时常维持一个不错的代码版本，用来回溯

我打算尝试第三次：希望能够顺利完成。