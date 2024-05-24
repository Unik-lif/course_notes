## Chapter 7
这一章我们着重研究一下`scheduler`等相关的切换机制。

我们直接来看，在哪些部分里头我们调用了`swtch`函数。
```C
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.	


.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret
```
`swtch`函数对于`callee-saved`寄存器进行了存储与读取工作，此外，特地修改了 `ra` 寄存器的值是为了在 `ret` 时能够正确跳转。需要注意的是，`caller-saved`寄存器会由编译器来自动进行存放和读取，所以作为写代码的人，其实只需要对相关的`callee-saved`寄存器做一些处理，就能完成`context`存储与拾取的工作。

首先是函数`scheduler`：
```C
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int nproc = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state != UNUSED) {
        nproc++;
      }
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        // 当前 scheduler 这边的上下文环境，存放到 c->context 之中
        // 随后，切换到 p->context 所对应的上下文环境
        // p 是遍历过来时，第一个是 RUNNABLE 的进程
        // 控制流从这边，彻底进入 p->context 的位置，对于一开始进入的 scheduler 函数，约等于我们的操作系统确实起来了
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
      }
      release(&p->lock);
    }
    // 优先将中断路由到当前的这个 hartid 所对应的 cpu 上
    if(nproc <= 2) {   // only init and sh exist
      intr_on();
      asm volatile("wfi");
    }
  }
}
```
上面的流程相对清楚，除了后面的 `wfi` ，其他部分都比较清楚。

第二个函数是 `sched`：该函数在 `yield`, `exit`, `sleep`等涉及挂起的函数中被使用
```C
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();
  // 首先，能够调用 sched 的进程必须能够先获得了自己的 p->lock
  if(!holding(&p->lock))
    panic("sched p->lock");
  // mycpu()->noff 为 1 ，表示当前所用的锁有且仅有 p->lock 一个
  if(mycpu()->noff != 1)
    panic("sched locks");
  // 检查进程 p->state 是否为 RUNNING , 为 RUNINNG 反而是奇怪的，因为我们需要切换到其他进程上
  if(p->state == RUNNING)
    panic("sched running");
  // 检查当前的 device interrupts 是否已经被开启了
  if(intr_get())
    panic("sched interruptible");

  // 是否已经 enable 了 interrupts
  intena = mycpu()->intena;
  // 切换到 mycpu()->context ，特别的，这个位置专门提供给 scheduler 函数
  swtch(&p->context, &mycpu()->context);
  // 下次重新切换回来的时候，还能继续恢复 intena 的值，因为 mycpu 的值可能会被切过去的其他线程所改变？
  mycpu()->intena = intena;
}
```
我们先前似乎对于 `pop_off` 以及 `push_off` 没有做较为详细的分析，现在分析如下：
```C
// push_off/pop_off are like intr_off()/intr_on() except that they are matched:
// it takes two pop_off()s to undo two push_off()s.  Also, if interrupts
// are initially off, then push_off, pop_off leaves them off.

void
push_off(void)
{
  int old = intr_get();
  // 总之在这边关闭掉了 intr_off 
  intr_off();
  // 如果当前 noff 中的值为 0 , 表示该 push_off 的深度为 0 ，这个 push_off 是第一层
  // 那么在 mycpu 中存放 intena 原本的值
  // 如果 pop_off 在之后有必要将 intr_on 重启，则第一层的该 old 值需要是 1
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}

void
pop_off(void)
{
  struct cpu *c = mycpu();
  // 在 pop_off 中，不应该看到 intr_get 为 1 ，它应该和 push_off 呼应，当 push_off 已经被调用了后， intr_get 的值就不可能是正数
  if(intr_get())
    panic("pop_off - interruptible");
  // 层数不可能比 1 还要小
  if(c->noff < 1)
    panic("pop_off");
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    intr_on();
}

// Per-CPU state.
struct cpu {
  struct proc *proc;          // The process running on this cpu, or null.
  struct context context;     // swtch() here to enter scheduler().
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};
```
到这边感觉应有的分析似乎都做好了，我们可以考虑做实验了。

### 第二部分
我们继续分析一些同步原语在这边的应用情况：首先是 sleep ，它的语义和 Unix 的基本维持一致。
```C
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  // 在调用 sleep 时，首先你得有 p->lock
  // 利用两个锁的重合部分，避免敏感区域攻击窗口
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.
  if(lk != &p->lock){  //DOC: sleeplock0
    acquire(&p->lock);  //DOC: sleeplock1
    release(lk);
  }

  // Go to sleep.
  // 设置当前进程的频段为 chan
  p->chan = chan;
  p->state = SLEEPING;
  // 调用到另外一个进程
  sched();
  // 回到这边的时候，已经 wakeup 了，p->state 也不是 SLEEPING
  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  // 结束 sleep 后，需要重新获取 lk ，不过为什么要释放 p->lock ?
  // 因为操作做完了，可以释放 p->lock 了
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}
```
我们再看向对应的 wakeup 函数：
```C
// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == SLEEPING && p->chan == chan) {
      p->state = RUNNABLE;
    }
    release(&p->lock);
  }
}
```
功能很简单，似乎就是先获取 p->lock ，对相关进程做遍历直到看到某个频段一致，再设置它为 RUNNABLE 状态。