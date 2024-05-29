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

在后面的章节针对 pipe 的读写做了一些简单的研究，我们调研如下：一般来说，我们先得 write ，才能 read
```C
int
pipewrite(struct pipe *pi, uint64 addr, int n)
{
  int i;
  char ch;
  struct proc *pr = myproc();

  acquire(&pi->lock);
  for(i = 0; i < n; i++){
    // 满了就不能再向这个buff空间中写了，那样的话就得进入 wakeup ，让 piperead 把东西消费掉
    // 然后自己陷入睡觉状态，直到 piperead 搞定后唤醒自己
    while(pi->nwrite == pi->nread + PIPESIZE){  //DOC: pipewrite-full
      if(pi->readopen == 0 || pr->killed){
        release(&pi->lock);
        return -1;
      }
      wakeup(&pi->nread);
      sleep(&pi->nwrite, &pi->lock);
    }
    // 需要通过页表进行解析得知 addr + i 对应的字符到底是什么
    if(copyin(pr->pagetable, &ch, addr + i, 1) == -1)
      break;
    // 然后再赋值给 data 部分
    pi->data[pi->nwrite++ % PIPESIZE] = ch;
  }
  wakeup(&pi->nread);
  release(&pi->lock);
  return i;
}

int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;
  // 首先尝试获得 pipe 所对应的 lock 结构
  acquire(&pi->lock);
  // 如果该 pipe 是空的，并且 pi->writeopen 端是开启着的
  // 说明有人打算向这个 pipe 进行写操作，但是目前这边还是空的
  // 自然，我们此时没有办法从 pipe 中读取字符
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(pr->killed){
      release(&pi->lock);
      return -1;
    }
    // 没办法，我们只好等待，等到 pi->nread 的值被唤醒
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  // 否则说明 pi->nread 和 pi->nwrite 的值不同
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    // 则从 pi->data 读取值出来，直到读完 n 个，或者读完 pipe buff 内存放的字符
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  // 告知 nwrite ，即 pipewrite 可以运行了
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```
pipe code 拥有独立的 sleep channels
### 系统调用的实现
Wait, exit and kill

wait 与 exit ，以及 kill 是工作在一起的

首先我们来看一下 wait 系统调用：
```C
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *np;
  int havekids, pid;
  struct proc *p = myproc();

  // hold p->lock for the whole time to avoid lost
  // wakeups from a child's exit().
  acquire(&p->lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    // 对于 np 进行遍历
    for(np = proc; np < &proc[NPROC]; np++){
      // this code uses np->parent without holding np->lock.
      // acquiring the lock first would cause a deadlock,
      // since np might be an ancestor, and we already hold p->lock.
      // 检测遍历过来是否正好是自己的孩子
      if(np->parent == p){
        // np->parent can't change between the check and the acquire()
        // because only the parent changes it, and we're the parent.

        // 必须等到 np 跑完后，父进程才能得到 np->lock
        acquire(&np->lock);
        havekids = 1;
        if(np->state == ZOMBIE){
          // Found one.
          pid = np->pid;
          // 将子进程的 status 记录到 addr 所对应的地址上
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,
                                  sizeof(np->xstate)) < 0) {
            release(&np->lock);
            release(&p->lock);
            return -1;
          }
          // 释放掉子进程的进程空间
          freeproc(np);
          release(&np->lock);
          release(&p->lock);
          return pid;
        }
        release(&np->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || p->killed){
      release(&p->lock);
      return -1;
    }
    
    // Wait for a child to exit.
    // 没跑完就等着
    sleep(p, &p->lock);  //DOC: wait-sleep
  }
}
```
我们再研究一下 kill：会发现它的结构比我们想象的简单很多，其实就只是获得 p->lock 之后，设置 p->killed = 1 之类的事情。不过为什么要把 p->state 从 SLEEPING 切换到 RUNNABLE？兴许是重新将其放到调度队列中，否则没法处理？
```C
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```
为了搞清楚这一点，以及与 wait 的联合使用，我们需要继续研究一下 exit ：
```C
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }
  // 涉及了部分文件系统相关的操作
  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  // we might re-parent a child to init. we can't be precise about
  // waking up init, since we can't acquire its lock once we've
  // acquired any other proc lock. so wake up init whether that's
  // necessary or not. init may miss this wakeup, but that seems
  // harmless.
  acquire(&initproc->lock);
  wakeup1(initproc);
  release(&initproc->lock);

  // grab a copy of p->parent, to ensure that we unlock the same
  // parent we locked. in case our parent gives us away to init while
  // we're waiting for the parent lock. we may then race with an
  // exiting parent, but the result will be a harmless spurious wakeup
  // to a dead or wrong process; proc structs are never re-allocated
  // as anything else.
  acquire(&p->lock);
  struct proc *original_parent = p->parent;
  release(&p->lock);
  
  // we need the parent's lock in order to wake it up from wait().
  // the parent-then-child rule says we have to lock it first.
  acquire(&original_parent->lock);

  acquire(&p->lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup1(original_parent);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&original_parent->lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```
到这边，我们对于这一章的分析算是结束了。不过似乎还不足以支撑我们继续往下做实验。