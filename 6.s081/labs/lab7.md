## 实验概览
第一个实验在`xv6`上做，二三两个实验我们在一般的`ubuntu`环境上做。
### uthread
尝试在用户层次设计线程系统。我们首先来看两个关键文件：`uthread.c`以及`uthread_switch.S`。
```C
int 
main(int argc, char *argv[]) 
{
  a_started = b_started = c_started = 0;
  a_n = b_n = c_n = 0;
  thread_init();
  thread_create(thread_a);
  thread_create(thread_b);
  thread_create(thread_c);
  thread_schedule();
  exit(0);
}
```
主程序通过`thread_init`做了一些简单的初始化，然后通过`thread_create`来创建`thread_a`到`thread_c`。
```C
void 
thread_init(void)
{
  // main() is thread 0, which will make the first invocation to
  // thread_schedule().  it needs a stack so that the first thread_switch() can
  // save thread 0's state.  thread_schedule() won't run the main thread ever
  // again, because its state is set to RUNNING, and thread_schedule() selects
  // a RUNNABLE thread.
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}
```
简单的工作，设置当前主线程的号为`0`，并告诉他自己的状态为 `RUNNING` 。
```C
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
}
```
暂不太清楚具体需要做什么，这个函数的框架是找到一个空闲的线程，然后设置他状态为 `RUNNABLE`。

三个线程行为都是一致的：检查其他线程是否还在跑，如果是，则先挂起一会儿。进去之后，从`0`到`100`进行打印，每次打印都挂起一次，然后再退出。退出的时候，利用`thread_schedule`切换到另外的线程上。
```C
void 
thread_a(void)
{
  int i;
  printf("thread_a started\n");
  a_started = 1;
  while(b_started == 0 || c_started == 0)
    thread_yield();
  
  for (i = 0; i < 100; i++) {
    printf("thread_a %d\n", i);
    a_n += 1;
    thread_yield();
  }
  printf("thread_a: exit after %d\n", a_n);

  current_thread->state = FREE;
  thread_schedule();
}
```
对于挂起和切换，简单分析如下所示：
```C
// 将当前线程的状态切换到 RUNNABLE ，并调用 thread_schedule
void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}

void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  // t 被设置为下一个线程
  for(int i = 0; i < MAX_THREAD; i++){
    // 越界则重新返回到第 0 个线程
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    // 如果 t 的状态为 RUNNABLE，则设置 next_thread 为 t
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }
  // 找不到任何一个状态为 RUNNABLE 的线程
  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }
  // current_thread == next_thread 说明没得换了，对应下面的 next_thread = 0
  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
  } else
    next_thread = 0;
}
```
到这边，我们似乎对框架有了大体的认知，此外还有一个用于切换上下文的函数 `thread_switch` ，它位于汇编文件中。
```S
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
	ret    /* return to ra */
```

有上面的寄出之后，对于写代码这件事情不会太难了。

一些关键点：
- 模仿先前`swtch`中的代码运转来写我们的`thread_switch`
- 模仿一开始进入线程时对于部分寄存器的设置
- 要合理设置栈的位置，否则可能会覆盖掉我们的关键寄存器的信息

感觉最关键的代码其实也就这两样：
```C
  // But we won't switched to the thread t here.
  *((uint64*)t->stack) = (uint64)func;
  *((uint64*)(t->stack + 8)) = (uint64)t->stack + 2048;
```
### using threads
我们需要对于程序做一个简单的研究，反正就是涉及敏感部分的操作，记得加锁。

然后按照`Morris`的方法论来做，从大锁缩小的小锁，一下子你就做出来了。
### barrier
思考清楚了后这件事还挺简单的：
```C
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread += 1;
  if(bstate.nthread == nthread){
    bstate.round += 1;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);     // wake up every thread sleeping on cond
  }else{
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```