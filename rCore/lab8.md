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
### 信号量
两种信号量声明：`P`操作和`V`操作。`P`操作是检查信号量的值是否大于`0`，如果大于`0`，则将他的值减`1`并且继续。如果这个值为`0`，则线程将睡眠。

在`P`操作中，检查/修改信号量值以及可能发生的睡眠这一系列操作， 是一个不可分割的原子操作过程。

`V`操作会对信号量的值加`1` ，然后检查是否有一个或多个线程在该信号量上睡眠等待。如有，则选择其中的一个线程唤醒并允许该线程继续完成它的`P`操作；如没有，则直接返回。

这两个操作的伪代码结构如下所示：其中`S`表示可以进入临界区的线程数目，当`S`取值为`1`，表示他是二值信号量，即为互斥锁。
```rust
fn P(S) {
    if S >= 1
        S = S - 1;
    else
        <block and enqueue the thread>;
}
fn V(S) {
    if <some threads are blocked on the queue>
        <unblock a thread>;
    else
        S = S + 1;
}
```
信号量的另外一种实现如下所示：
```rust
fn P(S) {
    S = S - 1;
    if S < 0 then
        <block and enqueue the thread>;
}

fn V(S) {
    S = S + 1;
    if <some threads are blocked on the queue>
        <unblock a thread>;
}
```
在这边的S的初始值一般为一个大于`0`的正整数，表示可以进入临界区的线程数，但是S的取值可以是小于`0`的，表示等待进入临界区的睡眠线程数。这种用途下的信号量可以用于同步，在我们把初始值设置为`0`时，就能体现出这个效果。
```rust
let static mut S: semaphore = 0;

//Thread A
...
P(S); // 显然此处的A会睡眠
Label_2:
A-stmts after Thread B::Label_1; 
...

//Thread B
...
B-stmts before Thread A::Label_2;
Label_1:
V(S); // B唤醒了A，以此可以保证B线程的运行在A线程运行之前
...
```
#### 信号量系统调用的实现：
信号量是用于线程之间同步用的，因此非常显然也是由我们的进程来进行管理。
```rust
    /// semaphore list
    pub semaphore_list: Vec<Option<Arc<Semaphore>>>,
```
信号量具体涉及的数据结构如下所示：
```rust
/// semaphore structure
pub struct Semaphore {
    /// semaphore inner
    pub inner: UPSafeCell<SemaphoreInner>,
}

pub struct SemaphoreInner {
    // 当前S的大小
    pub count: isize,
    // 当前被要求去休眠的线程们
    pub wait_queue: VecDeque<Arc<TaskControlBlock>>,
}

impl Semaphore {
    /// Create a new semaphore
    pub fn new(res_count: usize) -> Self {
        trace!("kernel: Semaphore::new");
        Self {
            inner: unsafe {
                UPSafeCell::new(SemaphoreInner {
                    count: res_count as isize,
                    wait_queue: VecDeque::new(),
                })
            },
        }
    }

    /// up operation of semaphore，即V操作
    pub fn up(&self) {
        trace!("kernel: Semaphore::up");
        let mut inner = self.inner.exclusive_access();
        // 给S加一
        inner.count += 1;
        // 如果当前的S还是小于等于0，说明临界区现在并没有什么线程正在运行，可以唤醒预约好的客人来
        // 需要从我们的等待队列唤醒一个线程
        if inner.count <= 0 {
            if let Some(task) = inner.wait_queue.pop_front() {
                wakeup_task(task);
            }
        }
    }

    /// down operation of semaphore
    pub fn down(&self) {
        trace!("kernel: Semaphore::down");
        let mut inner = self.inner.exclusive_access();
        // 给S减一
        inner.count -= 1;
        // 如果当前的S小于0，则说明来的不是时候，建议睡一会儿，一会儿有人完事儿了就打电话通知
        if inner.count < 0 {
            inner.wait_queue.push_back(current_task().unwrap());
            drop(inner);
            block_current_and_run_next();
        }
    }
}
```
这个模型是用于同步的，到底怎么操作得看用户程序具体的使用法则。
### 条件变量
条件变量是一个更高层次的同步原语，在有些情况下，线程需要检查某一条件满足之后，才会继续执行。

一般来说，在满足一些条件之后，线程需要配合互斥锁以较为正确地完成带条件的同步流程，如下所示：
```rust
static mut A: usize = 0;
unsafe fn first() -> ! {
    mutex.lock();
    A=1;
    mutex.unlock();
    ...
}

unsafe fn second() -> ! {
    mutex.lock();
    while A==0 {
        mutex.unlock();
        // give other thread a chance to lock
        mutex.lock();
    };
    mutex.unlock();
    //继续执行相关事务
}
```
当然，上述的实现同样存在着一定的问题，因为需要忙等的原因，我们的程序跑起来比较低效。不如让second函数休眠一会儿？
```rust
static mut A: usize = 0;
unsafe fn first() -> ! {
    mutex.lock();
    A=1;
    wakup(second);
    mutex.unlock();
    ...
}

unsafe fn second() -> ! {
    mutex.lock();
    while A==0 {
       wait();
    };
    mutex.unlock();
    //继续执行相关事务
}
```
然而，这样的执行方式同样是存在很大问题的，由于`second`是带着锁去睡觉的，`first`也同时拿不到锁，这样就出现死锁了，很不妙。

如何等待一个条件？如何在条件为真时向等待线程发出信号？
#### 基本思路：
管程：任意时刻只能有一个活跃线程调用管程中的过程，这一特性使得线程在调用执行管程中过程时能够保证互斥，线程就可以很放心地去访问共享变量。

管程是编程语言的组成部分，编译器知道它的特殊性，因此可以采用与其他过程调用不同的方法来处理对于管程的调用。

然而，管程本身还需要做一些规定：
- 等待机制：线程在调用管程中的某个过程中，发现某个条件不满足，那就无法继续运行而被阻塞。
- 唤醒机制：另外一个线程可以在调用管程的过程中，把某个条件设置为真。
- 还需要有一个机制：及时唤醒等待条件为真的阻塞线程。

在`rCore`实验中似乎更加倾向于采用`Hansen`语义：执行唤醒操作的线程必须立即退出管程，即唤醒操作只可能作为一个管程过程的最后一条语句。唤醒线程的执行位置也因此会在此时离开管程。

该方案的具体实现就是条件变量和它所对应的操作：`wait`和`signal`。线程使用条件变量来等待一个条件变成真。

条件变量本质上是一个线程等待队列，如果条件不满足，线程会通过`wait`操作把自己加入到等待队列中。此外，在某个线程改变条件为真之后，就可以通过条件变量的`signal`操作来唤醒一个或者多个等待的线程，通过在该条件上发相关的信号，让他们继续执行。
```rust
static mut A: usize = 0;
unsafe fn first() -> ! {
    mutex.lock();
    A=1;
    condvar.wakup();
    mutex.unlock();
    ...
}

unsafe fn second() -> ! {
    mutex.lock();
    while A==0 {
       condvar.wait(mutex); //在睡眠等待之前，需要释放mutex
    };
    mutex.unlock();
    //继续执行相关事务
}

fn wait(mutex) {
    mutex.unlock();
    <block and enqueue the thread>;// 挂起后就会陷在这个位置，并且自己会与条件变量绑定在一起
    mutex.lock();
}

fn signal() {
    <unblock a thread>; // 找到挂在条件变量上睡眠的线程，把它唤醒
}
```
一个简单的应用例子如下所示：
```rust
static mut A: usize = 0;   //全局变量

const CONDVAR_ID: usize = 0;
const MUTEX_ID: usize = 0;

unsafe fn first() -> ! {
    sleep(10);
    println!("First work, Change A --> 1 and wakeup Second");
    mutex_lock(MUTEX_ID);
    // 改变条件变量，以满足second的条件
    A=1;
    // 火速唤醒second线程
    condvar_signal(CONDVAR_ID);
    mutex_unlock(MUTEX_ID);
    ...
}
unsafe fn second() -> ! {
    println!("Second want to continue,but need to wait A=1");
    mutex_lock(MUTEX_ID);
    while A==0 {
        // 把自己和CONDVAR_ID绑定在一起
        // 释放mutex锁并且阻塞（不可以带着锁入睡，不然会死锁
        condvar_wait(CONDVAR_ID, MUTEX_ID);
    }
    mutex_unlock(MUTEX_ID);
    ...
}
pub fn main() -> i32 {
    // create condvar & mutex
    assert_eq!(condvar_create() as usize, CONDVAR_ID);
    assert_eq!(mutex_blocking_create() as usize, MUTEX_ID);
    // create first, second threads
    ...
}

pub fn condvar_create() -> isize {
    sys_condvar_create(0)
}
pub fn condvar_signal(condvar_id: usize) {
    sys_condvar_signal(condvar_id);
}
pub fn condvar_wait(condvar_id: usize, mutex_id: usize) {
    sys_condvar_wait(condvar_id, mutex_id);
}
```
#### 条件变量机制的系统调用实现
与上述一致，条件变量是由我们的进程来进行管理的。
```rust
/// Condition variable structure
pub struct Condvar {
    /// Condition variable inner
    pub inner: UPSafeCell<CondvarInner>,
}

pub struct CondvarInner {
    // 内部是线程的等待队列
    pub wait_queue: VecDeque<Arc<TaskControlBlock>>,
}

impl Condvar {
    /// Create a new condition variable
    pub fn new() -> Self {
        trace!("kernel: Condvar::new");
        Self {
            inner: unsafe {
                UPSafeCell::new(CondvarInner {
                    wait_queue: VecDeque::new(),
                })
            },
        }
    }

    /// Signal a task waiting on the condition variable
    // 唤醒一个线程，本质上其实就是从等待队列中把它pop出来
    pub fn signal(&self) {
        let mut inner = self.inner.exclusive_access();
        if let Some(task) = inner.wait_queue.pop_front() {
            wakeup_task(task);
        }
    }

    /// blocking current task, let it wait on the condition variable
    // 陷入等待时，要把锁释放，以防止带着锁睡觉进入死锁状态
    pub fn wait(&self, mutex: Arc<dyn Mutex>) {
        trace!("kernel: Condvar::wait_with_mutex");
        mutex.unlock();
        let mut inner = self.inner.exclusive_access();
        inner.wait_queue.push_back(current_task().unwrap());
        drop(inner);
        block_current_and_run_next();
        mutex.lock();
    }
}
```
到这边，我们全部的源码解读就完成了。什么，系统调用没解读？我想把上面的机制搞清楚后，看系统调用真的很简单了。
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
不过根据后来的一些测试，为了减少调试超时带来的麻烦，建议读者们把`get_sys_time`这个系统调用加上，其他的都可以不加。
### 思路：
这个死锁检测算法理解起来其实不难，但是如何把它做好呢？

这边的算法似乎是给`blocking mutex`准备的，不过这个不重要。为了解耦合，我们应该考虑最好在系统调用这个层面做一些简单的修改，并不把相关的修改内陷到锁这个东西本身上。

根据测试样例，基本上确定思路如下：
#### 第一步：分成两半
需要实现两个检测，其一是我们的锁的检测，这个实现起来目测不是很难，至少在`mutex`的检测上我们不需要套用在文档中提到的检测算法。具体来说，我们可以遍历一遍`mutex_list`，如果我们`mutex_id`是重复的，则返回错误。

如何检测mutex_id已经被使用了？这边或许可以考虑

当然，既然`enable`了死锁检测，我们需要在管理进程的`PCB`中添加一个项来说明我们确实添加了死锁检测。

第一步完成，我们已经通过了`mutex_list`的测试。
#### 第二步：semaphore测试
检测算法面向的对象其实是我们的信号量，阅读了一下测试样例，通过纸笔绘画，基本确定了这个流程到底是怎么样的。

考虑到`tid`的分配可能不是连续的，即`alloc_fd`可能不是递增分配的，个人感觉利用`BTreeMap`数据结构更加合适一些：
```rust
BTreeMap<tid, status>

status {
    need: Vec::new(),
    allocation: Vec::new(),
    finish: bool,
}
```
然而，这样操作还是比较麻烦。如果可以的话，我们还是尽可能直接用数组吧。其实在这边只要能过测试感觉就足够了？偷个懒吧。之后如果有兴趣我们再用可以推广的方法来做。-> 然而检测发现线程号确实不确定，最好还是用推广的方法来做吧。

因此下面的方案还是作废算了。
```rust
    /// Lab5: enable the deadlock detection.
    pub dead_detect: bool,
    /// Lab5: semaphore deadlock detection part.
    /// I am a little afraid that my heap will overflow...
    /// Lab5: available resources.
    /// assume we at most have 10 threads and 10 kinds of semaphores.
    pub available: [u32; 10],
    /// Lab5: allocation vector.
    pub allocation: [[u32; 10]; 10],
    /// Lab5: need vector.
    pub need: [[u32; 10]; 10],
    /// Lab5: work vector for resources.
    pub work: [u32; 10],
    /// Lab5: finish vector for threads.
    pub finish: [bool; 10],
```
死锁到底会不会发生不取决于我们在`TCB`内写入的这个算法，我的意思是，我们的调度工作和我们的判别工作本质上是独立的。判别工作只是判断在理论上我们的调度会不会出现问题，如果检测出问题能够提供我们一个快速退出的路径。

虽然我们的线程是在动态地添加自己所对应的锁资源，但这与我们的检测行为并不是冲突的。但是，每次检测我们或许都需要把原来的`finish`归为`false`？与其这样子做，不如我每次都重新建立一个局部变量来做，这样就会自动释放掉，不需要再管理了。
```rust
    /// Lab5: semaphore deadlock detection part.
    /// I am a little afraid that my heap will overflow...
    /// Lab5: available resources.
    /// assume we at most have 10 threads and 10 kinds of semaphores.
    pub available: Vec<usize>,
    /// Lab5: allocation vector.
    pub allocation: Vec<Vec<usize>>,
    /// Lab5: need vector.
    /// pub need: Vec<Vec<usize>>,
    /// Lab5: work vector for resources.
    pub work: Vec<usize>,
```
#### 第三步：我们需要怎么修改我们的参数？
据说这个算法叫做银行家算法。

为了实现银行家算法的检测，我们需要对三个函数都做一些有针对性的修改。

首先，在`create`函数中，我们获得了一些资源，为此需要更新`available`，由于id号的位置不确定，所以需要适时`extend`一下下。
```rust
    if process_inner.dead_detect {
        if id >= process_inner.available.len() {
            for _ in (process_inner.available.len() - 1)..id {
                process_inner.available.push(0);
            }
        }
        process_inner.available[id] += res_count;
    }
```
第二，在`semaphore_up`函数中，我们会从之前分配的单元中重新得到我们的资源，此外当前线程利用`need`所对应的资源量会适时地减少。

第三，在`semaphore_down`函数中，我们需要执行银行家算法的检测。

第一步，我们需要把`need`写进去，表示这个需求的存在，这时候再考虑我们当前的情况是否有可能发生死锁状况。(对吗？不太对)

第二步，我们需要遍历一遍我们的线程，检查每个线程所对应的资源是否可以被当前的`work`所满足。如果可以满足，我们回收相关的资源，把它写到一个`vec`中，存储它的`tid`信息。如果不能满足，我们继续往下遍历。

特别的，我们每次在执行`semaphore_down`时都会对死锁情况进行了一个检查，而`need`表示接下来仍然需要分配的东西，这也就意味着其实我们每次的检查都是很简单的：每一步`need`实际上只有一个数，且对应的是当前线程`tid`和当前信号量`sem_id`。

#### 重新整理一遍：
在`semaphore_up`的实现中，仍然存在`tid`超出上述数据集的可能性。因此第一步，我们还是要看一下`tid`是否超过了范围，没有超过的话最好，超过的话要重复先前的`extend`过程。

`work`其实是可以当做局部变量建立的，每次开始时均要和`available`保持一致。
```rust
    /// Lab5: semaphore deadlock detection part.
    /// I am a little afraid that my heap will overflow...
    /// Lab5: available resources.
    /// assume we at most have 10 threads and 10 kinds of semaphores.
    pub available: Vec<usize>,
    /// Lab5: allocation vector.
    pub allocation: Vec<Vec<usize>>,
    /// Lab5: need vector.
    pub need: Vec<Vec<usize>>,
    /// Lab5: work vector for resources.
    /// pub work: Vec<usize>,
    pub matrix: (usize, usize),
```
最后用这样的数据结构通关了。