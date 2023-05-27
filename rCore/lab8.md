## 第八章：并发
### 线程
线程是对进程结构的细化，进程则是线程的管理者。在同一个进程中的线程之间没有父子关系，大家都是兄弟，但是一开始通过`fork`创建的进程所对应的线程被称为线程的老大。

通过源码阅读发现进程的数据结构出现了一些变化：
```rust
/// Task control block structure
pub struct TaskControlBlock {
    /// immutable
    pub process: Weak<ProcessControlBlock>,
    /// Kernel stack corresponding to PID
    pub kstack: KernelStack,
    /// mutable
    inner: UPSafeCell<TaskControlBlockInner>,
}
```
虽然依旧采用了`TaskControlBlock`的组织方式，但`process`和`inner`的主要面向对象是不一样的。
```rust
/// Process Control Block
pub struct ProcessControlBlock {
    /// immutable
    pub pid: PidHandle,
    /// mutable
    inner: UPSafeCell<ProcessControlBlockInner>,
}

/// Inner of Process Control Block
pub struct ProcessControlBlockInner {
    /// is zombie?
    pub is_zombie: bool,
    /// memory set(address space)
    pub memory_set: MemorySet,
    /// parent process
    pub parent: Option<Weak<ProcessControlBlock>>,
    /// children process
    pub children: Vec<Arc<ProcessControlBlock>>,
    /// exit code
    pub exit_code: i32,
    /// file descriptor table
    pub fd_table: Vec<Option<Arc<dyn File + Send + Sync>>>,
    /// signal flags
    pub signals: SignalFlags,
    /// tasks(also known as threads)
    pub tasks: Vec<Option<Arc<TaskControlBlock>>>,
    /// task resource allocator
    pub task_res_allocator: RecycleAllocator,
    /// mutex list
    pub mutex_list: Vec<Option<Arc<dyn Mutex>>>,
    /// semaphore list
    pub semaphore_list: Vec<Option<Arc<Semaphore>>>,
    /// condvar list
    pub condvar_list: Vec<Option<Arc<Condvar>>>,
}
```
`ProcessControlBlock`本身更加侧重于从进程的这一视角去看问题，存放了进程的相关信息。
```rust
pub struct TaskControlBlockInner {
    pub res: Option<TaskUserRes>,
    /// The physical page number of the frame where the trap context is placed
    pub trap_cx_ppn: PhysPageNum,
    /// Save task context
    pub task_cx: TaskContext,

    /// Maintain the execution status of the current process
    pub task_status: TaskStatus,
    /// It is set when active exit or execution error occurs
    pub exit_code: Option<i32>,
}
/// User Resource for a task
pub struct TaskUserRes {
    /// task id
    pub tid: usize,
    /// user stack base
    pub ustack_base: usize,
    /// process belongs to
    pub process: Weak<ProcessControlBlock>,
}
```
`TaskControlBlockInner`则更加站在线程的角度，其中的`TaskUserRes`存放了线程关键信息，还存放了线程的用户栈之类的东东，并用弱引用的`process`为线程声明被所有权，即管理他的进程是谁以及当前的状况，如果进程已经寄了，那么弱引用最终可以通过判别得知，且能较为方便地利用`drop`进行释放。这里的思想和上一个实验的`pipebuffer`有异曲同工之妙。

然而这边的数据结构还是给了我一个问题，从数据结构上看不出一个进程管理存在多个线程的线程池这一架构，它到底是怎么组织的？

.....然后我发现我似乎搞错了，我们知道线程现在已经成了真正的最小单元，因此`Processor`处理的直接对象是线程，也就是说，上面的`task`才是线程单元，`TaskControlBlock`反映了这个线程单元，之所以`process`是个弱引用，想必到这边也很清楚了。那么，我们再在`PCB`之中看到了一个存放`Tasks`的大数组，好吧，这件事情确实如此，确实有一定的迷惑性存在。
#### 线程的创建
参考系统调用`sys_thread_create`，会在当前进程之中开一个线程出来。

线程与进程拥有相同的地址空间，不需要做太多的变化，不过需要让内核为自己添加专用的用户态栈，以及，为了独立完成任务，还需要保留进程所拥有的`TRAMPOLINE`以实现跳转。
```rust
/// thread create syscall
pub fn sys_thread_create(entry: usize, arg: usize) -> isize {
    // 这个创建线程的行为是哪个线程干的？
    trace!(
        "kernel:pid[{}] tid[{}] sys_thread_create",
        // current_task反映的其实是线程，由于是弱引用，需要升格
        current_task().unwrap().process.upgrade().unwrap().getpid(),
        // tid确实存放在res之中
        current_task()
            .unwrap()
            .inner_exclusive_access()
            .res
            .as_ref()
            .unwrap()
            .tid
    );
    let task = current_task().unwrap();
    let process = task.process.upgrade().unwrap();
    // create a new thread
    // 需要看一下是怎么初始化的
    let new_task = Arc::new(TaskControlBlock::new(
        Arc::clone(&process),
        // 玄学？为什么和当前的task用的ustack_base是一样的？这其实是不应该的
        task.inner_exclusive_access()
            .res
            .as_ref()
            .unwrap()
            .ustack_base,
        true,
    ));
    // add new task to scheduler
    add_task(Arc::clone(&new_task));
    let new_task_inner = new_task.inner_exclusive_access();
    let new_task_res = new_task_inner.res.as_ref().unwrap();
    let new_task_tid = new_task_res.tid;
    let mut process_inner = process.inner_exclusive_access();
    // add new thread to current process
    let tasks = &mut process_inner.tasks;
    while tasks.len() < new_task_tid + 1 {
        tasks.push(None);
    }
    tasks[new_task_tid] = Some(Arc::clone(&new_task));
    let new_task_trap_cx = new_task_inner.get_trap_cx();
    *new_task_trap_cx = TrapContext::app_init_context(
        entry,
        new_task_res.ustack_top(),
        kernel_token(),
        new_task.kstack.get_top(),
        trap_handler as usize,
    );
    (*new_task_trap_cx).x[10] = arg;
    new_task_tid as isize
}
// --------------------
// 线程块的初始化：
impl TaskControlBlock {
    /// Create a new task
    pub fn new(
        // 进程管理者是谁
        process: Arc<ProcessControlBlock>,
        ustack_base: usize,
        alloc_user_res: bool,
    ) -> Self {
        let res = TaskUserRes::new(Arc::clone(&process), ustack_base, alloc_user_res);
        let trap_cx_ppn = res.trap_cx_ppn();
        let kstack = kstack_alloc();
        let kstack_top = kstack.get_top();
        Self {
            process: Arc::downgrade(&process),
            kstack,
            inner: unsafe {
                UPSafeCell::new(TaskControlBlockInner {
                    res: Some(res),
                    trap_cx_ppn,
                    task_cx: TaskContext::goto_trap_return(kstack_top),
                    task_status: TaskStatus::Ready,
                    exit_code: None,
                })
            },
        }
    }
}
```

## 实验：
先尝试考虑把前一部分移植吧，我们需要完成`6b`对应的基本移植工作。

首先我们仅仅把`sys_get_time`移植了，看看效果：
```shell
[PASS] found <sync_sem passed111042126941886!>
[PASS] found <Test write B OK111042126941886!>
[FAIL] not found <deadlock test mutex 1 OK111042126941886!>
[PASS] found <Test power_7 OK111042126941886!>
[PASS] found <forktest pass.>
[PASS] found <pipetest passed111042126941886!>
[PASS] found <race adder using spin mutex test passed111042126941886!>
[PASS] found <file_test passed111042126941886!>
[PASS] found <hello child process!>
[PASS] found <philosopher dining problem with mutex test passed111042126941886!>
[PASS] found <threads with arg test passed111042126941886!>
[PASS] found <exit pass.>
[PASS] found <Hello, world from user mode program!>
[PASS] found <test_condvar passed111042126941886!>
[PASS] found <child process pid = (\d+), exit code = (\d+)>
[PASS] found <Test write C OK111042126941886!>
[PASS] found <threads test passed111042126941886!>
[PASS] found <deadlock test semaphore 2 OK111042126941886!>
[PASS] found <mpsc_sem passed111042126941886!>
[FAIL] not found <deadlock test semaphore 1 OK111042126941886!>
[PASS] found <Test write A OK111042126941886!>
[PASS] found <Test power_5 OK111042126941886!>
[PASS] found <Test power_3 OK111042126941886!>
[PASS] not found <FAIL: T.T>
[PASS] not found <Test sbrk failed!>
```
如果我们把`sys_get_time`去掉，得到的结果是：
```shell
[PASS] found <Test power_3 OK5829231262111042126941886!>
[PASS] found <pipetest passed5829231262111042126941886!>
[PASS] found <test_condvar passed5829231262111042126941886!>
[FAIL] not found <deadlock test mutex 1 OK5829231262111042126941886!>
[PASS] found <race adder using spin mutex test passed5829231262111042126941886!>
[PASS] found <threads with arg test passed5829231262111042126941886!>
[PASS] found <exit pass.>
[PASS] found <Test write C OK5829231262111042126941886!>
[PASS] found <sync_sem passed5829231262111042126941886!>
[PASS] found <philosopher dining problem with mutex test passed5829231262111042126941886!>
[PASS] found <hello child process!>
[PASS] found <threads test passed5829231262111042126941886!>
[PASS] found <forktest pass.>
[FAIL] not found <deadlock test semaphore 1 OK5829231262111042126941886!>
[PASS] found <Test power_7 OK5829231262111042126941886!>
[PASS] found <file_test passed5829231262111042126941886!>
[PASS] found <mpsc_sem passed5829231262111042126941886!>
[PASS] found <Test write B OK5829231262111042126941886!>
[PASS] found <Hello, world from user mode program!>
[PASS] found <Test write A OK5829231262111042126941886!>
[PASS] found <Test power_5 OK5829231262111042126941886!>
[FAIL] not found <deadlock test semaphore 2 OK5829231262111042126941886!>
[PASS] found <child process pid = (\d+), exit code = (\d+)>
[PASS] not found <FAIL: T.T>
[PASS] not found <Test sbrk failed!>
```