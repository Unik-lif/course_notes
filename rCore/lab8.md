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
        // 玄学？为什么和当前的task用的ustack_base是一样的？
        // 或许就是这样？我可能得多去翻一些资料？
        // 5月28日答复：其实并不是，这只是一个基地址而已，方便你定位用的
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
    // 在当前的PCB之中写入我们的线程的信息
    let tasks = &mut process_inner.tasks;
    while tasks.len() < new_task_tid + 1 {
        tasks.push(None);
    }
    tasks[new_task_tid] = Some(Arc::clone(&new_task));
    let new_task_trap_cx = new_task_inner.get_trap_cx();
    // 
    *new_task_trap_cx = TrapContext::app_init_context(
        entry,
        new_task_res.ustack_top(),
        kernel_token(),
        new_task.kstack.get_top(),
        trap_handler as usize,
    );
    // 线程参数输入之，考虑到了命令行参数之类的东西
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
        let res = TaskUserRes::new(Arc::clone(&process), ustack_base, alloc_user_res); // res看样子用的参数都是一致的
        let trap_cx_ppn = res.trap_cx_ppn();
        let kstack = kstack_alloc(); // 内核栈需要内核进行分配，各自都是不一样的
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
// ----------------------
// res的初始化：
// 突出一个 确实 
    /// Create a new TaskUserRes (Task User Resource)
    pub fn new(
        process: Arc<ProcessControlBlock>,
        ustack_base: usize,
        alloc_user_res: bool,
    ) -> Self {
        // tid是自动分配的，至少在同一个process中不会重合
        let tid = process.inner_exclusive_access().alloc_tid();
        let task_user_res = Self {
            tid,
            ustack_base,
            process: Arc::downgrade(&process),
        };
        if alloc_user_res {
            task_user_res.alloc_user_res();
        }
        task_user_res
    }
    /// Allocate user resource for a task
    /// 基地址是在这边用的，而且会根据tid的具体情况来做
    pub fn alloc_user_res(&self) {
        let process = self.process.upgrade().unwrap();
        let mut process_inner = process.inner_exclusive_access();
        // alloc user stack
        // 特别的，会根据tid值来决定真正的用户栈地址
        let ustack_bottom = ustack_bottom_from_tid(self.ustack_base, self.tid);
        let ustack_top = ustack_bottom + USER_STACK_SIZE;
        process_inner.memory_set.insert_framed_area(
            ustack_bottom.into(),
            ustack_top.into(),
            MapPermission::R | MapPermission::W | MapPermission::U,
        );
        // alloc trap_cx
        // 不同的tid会对应不同的trap_cx位置，需要把他们存到相应的地址空间处
        let trap_cx_bottom = trap_cx_bottom_from_tid(self.tid);
        let trap_cx_top = trap_cx_bottom + PAGE_SIZE;
        process_inner.memory_set.insert_framed_area(
            trap_cx_bottom.into(),
            trap_cx_top.into(),
            MapPermission::R | MapPermission::W,
        );
    }
// -------
// 初始化中断上下文：
    /// init the trap context of an application
    pub fn app_init_context(
        entry: usize,
        sp: usize,
        kernel_satp: usize,
        kernel_sp: usize,
        trap_handler: usize,
    ) -> Self {
        let mut sstatus = sstatus::read();
        // set CPU privilege to User after trapping back
        sstatus.set_spp(SPP::User);
        let mut cx = Self {
            x: [0; 32],
            sstatus,
            sepc: entry,  // entry point of app
            kernel_satp,  // addr of page table
            kernel_sp,    // kernel stack
            trap_handler, // addr of trap_handler function
        };
        cx.set_sp(sp); // app's user stack pointer
        cx // return initial Trap Context of app
    }
```
#### 线程等待
```rust
/// wait for a thread to exit syscall
///
/// thread does not exist, return -1
/// thread has not exited yet, return -2
/// otherwise, return thread's exit code
pub fn sys_waittid(tid: usize) -> i32 {
    // 但本质上没干什么，等待tid对应的线程自行退出，然后需要在进程级别利用exit_code和tid信息进行同步
    // 主要还是线程结束后我们进行一个资源上的回收工作
    trace!(
        "kernel:pid[{}] tid[{}] sys_waittid",
        current_task().unwrap().process.upgrade().unwrap().getpid(),
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
    let task_inner = task.inner_exclusive_access();
    let mut process_inner = process.inner_exclusive_access();
    // a thread cannot wait for itself，确实
    if task_inner.res.as_ref().unwrap().tid == tid {
        return -1;
    }
    let mut exit_code: Option<i32> = None;
    // 我们真正需要等的线程是waited_task
    let waited_task = process_inner.tasks[tid].as_ref();
    if let Some(waited_task) = waited_task {
        if let Some(waited_exit_code) = waited_task.inner_exclusive_access().exit_code {
            exit_code = Some(waited_exit_code);
        }
    } else {
        // waited thread does not exist
        return -1;
    }
    if let Some(exit_code) = exit_code {
        // dealloc the exited thread
        process_inner.tasks[tid] = None;
        exit_code
    } else {
        // waited thread has not exited
        -2
    }
}
```
#### 进程相关系统调用的线程视角-简单浏览一下就好了
确实没有太大的改动，你说的对
### 锁
对于进程来说，锁本身是一个被其管理起来的一个资源。`mutex_list`表示的是实现了`Mutex`特性的物件的管理向量，存放在进程控制块中。
```rust
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
名为`Mutex`的特性主要有两个函数需要支持：其一是`lock`，其二是`unlock`。
```rust
/// Mutex trait
pub trait Mutex: Sync + Send {
    /// Lock the mutex
    fn lock(&self);
    /// Unlock the mutex
    fn unlock(&self);
}
```
在源代码中提供了两种锁的实现方式，其一为自旋锁，其二我们一会儿再说：
```rust
/// Spinlock Mutex struct
pub struct MutexSpin {
    locked: UPSafeCell<bool>,
}
// 自旋锁的建立本质上只需要看内部存放的bool类型是true还是false
impl MutexSpin {
    /// Create a new spinlock mutex
    pub fn new() -> Self {
        Self {
            locked: unsafe { UPSafeCell::new(false) },
        }
    }
}

impl Mutex for MutexSpin {
    /// Lock the spinlock mutex
    // 如果需要锁上这个自旋锁，我们需要轮询访问当前这个锁的状态
    // 如果锁现在还是true，表明这个锁尚在被其他的线程所使用，一般情况下我们会倾向于忙等，
    // 但是在考虑性能的情况下，需要suspend我们当前的线程先做其他的事情，直到下一个线程干好了后
    // 再回来进行轮询
    // 直到我们的lock函数最终获得了这个锁的所有权
    fn lock(&self) {
        trace!("kernel: MutexSpin::lock");
        loop {
            let mut locked = self.locked.exclusive_access();
            if *locked {
                drop(locked);
                suspend_current_and_run_next();
                continue;
            } else {
                *locked = true;
                return;
            }
        }
    }
    // unlock相比而言就很简单了，我们只要把lock设置为false状态，表示没有人占用了就行了
    fn unlock(&self) {
        trace!("kernel: MutexSpin::unlock");
        let mut locked = self.locked.exclusive_access();
        *locked = false;
    }
}
```
第二种锁的建立方式可能没有自旋锁那么直接，其维护了一个队列用于管理就绪的线程，并把锁分配给他们，以实现同步任务：
```rust
/// Blocking Mutex struct
// 本质上是一个阻塞锁
pub struct MutexBlocking {
    inner: UPSafeCell<MutexBlockingInner>,
}

// 阻塞锁的内部除了自身当前的锁情况bool类型，还有一个等待队列，存放着等待获得锁的线程信息们
pub struct MutexBlockingInner {
    locked: bool,
    wait_queue: VecDeque<Arc<TaskControlBlock>>,
}

impl MutexBlocking {
    /// Create a new blocking mutex
    // 在锁块的建立过程中，一开始锁的状态被设置为没有锁上，并且等待队列也还是空的
    pub fn new() -> Self {
        trace!("kernel: MutexBlocking::new");
        Self {
            inner: unsafe {
                UPSafeCell::new(MutexBlockingInner {
                    locked: false,
                    wait_queue: VecDeque::new(),
                })
            },
        }
    }
}

impl Mutex for MutexBlocking {
    /// lock the blocking mutex
    // 如果要获取锁，和自旋锁很类似，如果当前的锁无法获取，我们的阻塞锁会把这个task放到等待队列中，在让相关的线程进入休眠状态的同时，继续跑其他的线程
    // 然而，我们这个时候确实不需要忙等了。只要在unlock的时候把锁再送出去就好了，送给谁呢？那得看哪个小可爱恰恰好用lock函数搞到了锁了呢
    // 具体来说这里的阻塞仅是把task状态设置为block罢了，这种情况下不可能会被processor抓走拿去调度的
    fn lock(&self) {
        trace!("kernel: MutexBlocking::lock");
        let mut mutex_inner = self.inner.exclusive_access();
        if mutex_inner.locked {
            mutex_inner.wait_queue.push_back(current_task().unwrap());
            drop(mutex_inner);
            block_current_and_run_next();
        } else {
            mutex_inner.locked = true;
        }
    }

    /// unlock the blocking mutex
    // 这边似乎也没什么太难理解的，如果你用了unlock，前提是这个锁确实还是lock状态下的，所以这边添加了一层判别
    // 我们会从阻塞的等待队列中pop_front出第一个线程，这里的调度遵循FIFO，把它唤醒就好了
    fn unlock(&self) {
        trace!("kernel: MutexBlocking::unlock");
        let mut mutex_inner = self.inner.exclusive_access();
        assert!(mutex_inner.locked);
        if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
            wakeup_task(waking_task);
        } else {
            mutex_inner.locked = false;
        }
    }
}
```
#### 锁的系统调用实现：
锁的调度在这边似乎还是需要采用系统调用来实现：首先，来看一下`mutex`是如何被创建的。其中的`blocking`反映了创建锁的类型。
```rust
/// mutex create syscall
pub fn sys_mutex_create(blocking: bool) -> isize {
    trace!(
        "kernel:pid[{}] tid[{}] sys_mutex_create",
        current_task().unwrap().process.upgrade().unwrap().getpid(),
        current_task()
            .unwrap()
            .inner_exclusive_access()
            .res
            .as_ref()
            .unwrap()
            .tid
    );
    let process = current_process();
    let mutex: Option<Arc<dyn Mutex>> = if !blocking {
        Some(Arc::new(MutexSpin::new()))
    } else {
        Some(Arc::new(MutexBlocking::new()))
    };
    let mut process_inner = process.inner_exclusive_access();
    // 当前进程所管理的mutex_list中，是否已经有了锁，但是它现在的状态是none?
    if let Some(id) = process_inner
        .mutex_list
        .iter()
        .enumerate() 
        .find(|(_, item)| item.is_none())
        .map(|(id, _)| id)
    {
        // 有这样的锁的话，就直接送给他吧，虽然我也不知道什么情况下锁会变成none.
        process_inner.mutex_list[id] = mutex;
        id as isize
    } else {
        // 当前进程所管理的mutex_list中一个锁都不存在，或者所有的锁都不是none的状态，那这就没什么好评价的
        process_inner.mutex_list.push(mutex);
        process_inner.mutex_list.len() as isize - 1
    }
}
```
其次，来看一下锁上锁是怎么一回事：好像也不是什么回事，说白了就是调用了一下我们上面分析过的接口罢了。
```rust
/// mutex lock syscall
pub fn sys_mutex_lock(mutex_id: usize) -> isize {
    trace!(
        "kernel:pid[{}] tid[{}] sys_mutex_lock",
        current_task().unwrap().process.upgrade().unwrap().getpid(),
        current_task()
            .unwrap()
            .inner_exclusive_access()
            .res
            .as_ref()
            .unwrap()
            .tid
    );
    let process = current_process();
    let process_inner = process.inner_exclusive_access();
    let mutex = Arc::clone(process_inner.mutex_list[mutex_id].as_ref().unwrap());
    drop(process_inner);
    drop(process);
    mutex.lock();
    0
}
/// mutex unlock syscall
pub fn sys_mutex_unlock(mutex_id: usize) -> isize {
    trace!(
        "kernel:pid[{}] tid[{}] sys_mutex_unlock",
        current_task().unwrap().process.upgrade().unwrap().getpid(),
        current_task()
            .unwrap()
            .inner_exclusive_access()
            .res
            .as_ref()
            .unwrap()
            .tid
    );
    let process = current_process();
    let process_inner = process.inner_exclusive_access();
    let mutex = Arc::clone(process_inner.mutex_list[mutex_id].as_ref().unwrap());
    drop(process_inner);
    drop(process);
    mutex.unlock();
    0
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