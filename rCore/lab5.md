## 第五章：进程管理
### 进程概念
进程是在操作系统管理下的程序的一次执行过程。当一个应用的源程序被编译器成功构建之后，它会从源代码变成某种格式的可执行文件。

我们在上一章看到`ELF`文件内确实有多个逻辑段，并且利用了函数`from_elf`实现了装载，然而，静态的东东并不是真实进程的存在形式，如`ELF`这样的可执行文件的特色其实是其可以被内核加载并且执行，这就需要消耗掉一部分的硬件资源。

进程本质上是操作系统在选取某一个可执行文件并对其进行一次动态执行的过程，相比可执行文件，它的动态性主要体现在两点：
1. 它是一个有始有终的过程。
2. 在过程中对于可执行文件中给出的需求要相应对硬件/虚拟资源进行动态绑定和解绑。

因此，即便两个进程选择同一个可执行文件执行，他们也会是截然不同的进程，它们的启动时间、占据的硬件资源、输入数据均有可能有较大的差异。为了记录不同进程的资源占用情况，在内核中我们需要有一个进程管理器，它要有能力管理多个进程。
### 重要系统屌用
#### fork系统调用
在内核初始化完毕后会创建一个进程，即用户初始进程，它是内核中以硬编码方式创建的唯一一个进程，其他所有进程都是通过`fork`系统调用创造而来。
#### Wait系统调用
进程通过`exit`系统调用退出之后，它所占用的资源并不能够立刻被全部回收，比如该进程的内核栈目前就正用来进行系统调用处理。如何正确处理系统调用？一个典型的做法是当进程退出之后，让内核立即回收一部分资源并且将这个进程标记为僵尸进程。

之后，由该进程的父进程通过一个名为`waitpid`的系统调用来收集该进程的返回状态并且回收掉它所占据的全部资源，这样这个进程才被彻底销毁。
#### exec系统调用
`exec`系统调用用来执行不同的可执行文件。

`Unix`之所以将创建进程分成两个系统调用的组合过程，主要还是因为当时的内存空间比较小，不太好支持。此外，这样的拆分也让灵活性变得很强。

## 源码分析
这一章的基本的框架代码是一致的，但添加了`task`与`loader`的新板块。
```rust
pub fn rust_main() -> ! {
    clear_bss();
    kernel_log_info();
    mm::init();
    mm::remap_test();
    task::add_initproc();
    println!("after initproc!");
    trap::init();
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    loader::list_apps();
    task::run_tasks();
    panic!("Unreachable in rust_main!");
}
```
### task::add_initproc:
我们尝试对`task`模块的初始化做更多的解析
```rust
///Add init process to the manager
pub fn add_initproc() {
    add_task(INITPROC.clone());
}
```
这里可以看到调用了`INITPROC`这个全局变量一样的东西，`add_task`是对其的操作函数。
```rust
/// Add process to ready queue
pub fn add_task(task: Arc<TaskControlBlock>) {
    //trace!("kernel: TaskManager::add_task");
    TASK_MANAGER.exclusive_access().add(task);
}
```
函数`add_task`中蕴含了`TASK_MANAGER`这一全局变量，其初始化与相关操作流程如下所示：
```rust
///A array of `TaskControlBlock` that is thread-safe
/// TaskManager里头存储的是准备完毕状态的进程控制块，以Arc形式实现原子多读写，以VecDeque方式来存储。
pub struct TaskManager {
    ready_queue: VecDeque<Arc<TaskControlBlock>>,
}

/// A simple FIFO scheduler.
/// 这确实是一个FIFO调度队列，直接用了VecDeque的相关接口。
impl TaskManager {
    ///Creat an empty TaskManager
    pub fn new() -> Self {
        Self {
            ready_queue: VecDeque::new(),
        }
    }
    /// Add process back to ready queue
    pub fn add(&mut self, task: Arc<TaskControlBlock>) {
        self.ready_queue.push_back(task);
    }
    /// Take a process out of the ready queue
    pub fn fetch(&mut self) -> Option<Arc<TaskControlBlock>> {
        self.ready_queue.pop_front()
    }
}

/// UPSafeCell我们已经看过多次了，防止重复获取，也提供了内部可变性
lazy_static! {
    /// TASK_MANAGER instance through lazy_static!
    pub static ref TASK_MANAGER: UPSafeCell<TaskManager> =
        unsafe { UPSafeCell::new(TaskManager::new()) };
}
```
到这里`add_task`函数的功能明确了，`INITPROC`全局变量仍需要研究，而它应该对应着`Arc<TaskControlBlock>`这一类型，事实也确实如此。
```rust
lazy_static! {
    /// Creation of initial process
    ///
    /// the name "initproc" may be changed to any other app name like "usertests",
    /// but we have user_shell, so we don't need to change it.
    pub static ref INITPROC: Arc<TaskControlBlock> = Arc::new(TaskControlBlock::new(
        get_app_data_by_name("ch5b_initproc").unwrap()
    ));
}
```
此处的`TaskControlBlock`类型有了一些细微的变化，从任务到进程，我们看到其需要设置一个进程号，内核栈是上一章就有的东西，而`TaskControlBlockInner`则是对下面一部分数据结构的封装。
```rust
/// Task control block structure
///
/// Directly save the contents that will not change during running
pub struct TaskControlBlock {
    // Immutable
    /// Process identifier
    pub pid: PidHandle,

    /// Kernel stack corresponding to PID
    pub kernel_stack: KernelStack,

    /// Mutable
    inner: UPSafeCell<TaskControlBlockInner>,
}
```
我们确乎在这里看到了很多老朋友，不过需要注意的是`exit_code`与`parent`和`children`是新添加的信息，由于进程只有一个父亲，多个孩子，所以这样的设计是合理的。此外，还利用了一层UPSafeCell的封装，这也意味着如果我们要访问`TaskControlBlockInner`内部的一些信息，还需要一次`exclusive_access`。
```rust
pub struct TaskControlBlockInner {
    /// The physical page number of the frame where the trap context is placed
    pub trap_cx_ppn: PhysPageNum,

    /// Application data can only appear in areas
    /// where the application address space is lower than base_size
    pub base_size: usize,

    /// Save task context
    pub task_cx: TaskContext,

    /// Maintain the execution status of the current process
    pub task_status: TaskStatus,

    /// Application address space
    pub memory_set: MemorySet,

    /// Parent process of the current process.
    /// Weak will not affect the reference count of the parent
    pub parent: Option<Weak<TaskControlBlock>>,

    /// A vector containing TCBs of all child processes of the current process
    pub children: Vec<Arc<TaskControlBlock>>,

    /// It is set when active exit or execution error occurs
    pub exit_code: i32,

    /// Heap bottom
    pub heap_bottom: usize,

    /// Program break
    pub program_brk: usize,
}
```
那么填充这些进程控制块的信息是什么，此处调用了函数`get_app_data_by_name`，我们继续解读：
```rust
#[allow(unused)]
///get app data from name
pub fn get_app_data_by_name(name: &str) -> Option<&'static [u8]> {
    let num_app = get_num_app();
    (0..num_app)
        .find(|&i| APP_NAMES[i] == name)
        .map(get_app_data)
}
```
可以看到相关的信息似乎是从`get_num_app`函数中装在了进来。这个函数同样也是一个老生常谈的函数，会从`.S`文件中找到`_num_app`的位置，然后阅读包括文件数目和地址的信息，此外还有各个应用程序的名字信息。
```rust

    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad 42
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start

    ...............
    .global _app_names
_app_names:
    .string "ch2b_bad_address"
    .string "ch2b_bad_instructions"
    .string "ch2b_bad_register"
    .string "ch2b_hello_world"
    .string "ch2b_power_3"
    .string "ch2b_power_5"
    .string "ch2b_power_7"
    .string "ch3_sleep"
    .string "ch3_sleep1"
    .string "ch3_taskinfo"
    .string "ch3b_yield0"
    .string "ch3b_yield1"
    .string "ch3b_yield2"
    .string "ch4_mmap0"
    .string "ch4_mmap1"
    .string "ch4_mmap2"
    .string "ch4_mmap3"
    .string "ch4_unmap"
    .string "ch4_unmap2"
    .string "ch4b_sbrk"
    .string "ch5_exit0"
    .string "ch5_exit1"
```
在这边对`APP_NAMES`做了简单的声明，其利用`_app_names`将先前`link_app.S`中的应用文件名字全部获取，存放到自己的`Vec`之中。
```rust
lazy_static! {
    ///All of app's name
    static ref APP_NAMES: Vec<&'static str> = {
        let num_app = get_num_app();
        extern "C" {
            fn _app_names();
        }
        let mut start = _app_names as usize as *const u8;
        let mut v = Vec::new();
        unsafe {
            for _ in 0..num_app {
                let mut end = start;
                while end.read_volatile() != b'\0' {
                    end = end.add(1);
                }
                let slice = core::slice::from_raw_parts(start, end as usize - start as usize);
                let str = core::str::from_utf8(slice).unwrap();
                v.push(str);
                start = end.add(1);
            }
        }
        v
    };
}
```
利用`find`函数在`APP_NAMES`中找到了我们想要的应用对应的序号`app_id`之后，回到`link_app.S`存储应用程序地址的位置，利用下面的函数进行应用程序`ELF`的装载任务。
```rust
/// get applications data
pub fn get_app_data(app_id: usize) -> &'static [u8] {
    extern "C" {
        fn _num_app();
    }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
    assert!(app_id < num_app);
    unsafe {
        core::slice::from_raw_parts(
            app_start[app_id] as *const u8,
            app_start[app_id + 1] - app_start[app_id],
        )
    }
}
```
应用程序`ch5b_initproc`的特殊之处在于其模拟了一般操作系统的第一个进程的运行模式，之后系统将会打开`shell`来跑其他的应用程序，这确实还挺酷的。
### loader::list_apps:
一个异常`trivial`的打印函数，没什么好评价的。
```rust
///list all apps
pub fn list_apps() {
    println!("/**** APPS ****");
    for app in APP_NAMES.iter() {
        println!("{}", app);
    }
    println!("**************/");
}
```
### task::run_tasks:
这个函数包含着本章设计的核心思想，我们需要比较谨慎地对待它：
```rust
///The main part of process execution and scheduling
///Loop `fetch_task` to get the process that needs to run, and switch the process through `__switch`
pub fn run_tasks() {
    // 这个loop很有意思，没他这游戏只能开一把
    // schedule跑完后返回会重新返回到这边来。
    loop {
        let mut processor = PROCESSOR.exclusive_access();
        if let Some(task) = fetch_task() {
            let idle_task_cx_ptr = processor.get_idle_task_cx_ptr();
            // access coming task TCB exclusively
            // 本质上是从TASK_MANAGER中搞来一个task，然后再把它压入到我们的processor之中
            let mut task_inner = task.inner_exclusive_access(); // 调用inner内部的TaskControlBlockInner这一可变引用
            let next_task_cx_ptr = &task_inner.task_cx as *const TaskContext;
            task_inner.task_status = TaskStatus::Running;
            // release coming task_inner manually
            drop(task_inner);
            // release coming task TCB manually
            processor.current = Some(task);
            // release processor manually
            drop(processor);
            unsafe {
                __switch(idle_task_cx_ptr, next_task_cx_ptr);
            }
        } else {
            warn!("no tasks available in run_tasks");
        }
    }
}
```
首先，何为`PROCESSOR`？
```rust
/// Processor management structure
pub struct Processor {
    ///The task currently executing on the current processor
    current: Option<Arc<TaskControlBlock>>,

    ///The basic control flow of each core, helping to select and switch process
    idle_task_cx: TaskContext,
}

impl Processor {
    ///Create an empty Processor
    pub fn new() -> Self {
        Self {
            current: None,
            idle_task_cx: TaskContext::zero_init(),
        }
    }

    ///Get mutable reference to `idle_task_cx`
    fn get_idle_task_cx_ptr(&mut self) -> *mut TaskContext {
        &mut self.idle_task_cx as *mut _
    }

    ///Get current task in moving semanteme
    pub fn take_current(&mut self) -> Option<Arc<TaskControlBlock>> {
        self.current.take()
    }

    ///Get current task in cloning semanteme
    pub fn current(&self) -> Option<Arc<TaskControlBlock>> {
        self.current.as_ref().map(Arc::clone)
    }
}

lazy_static! {
    pub static ref PROCESSOR: UPSafeCell<Processor> = unsafe { UPSafeCell::new(Processor::new()) };
}
```
可以看到`Processor`内管理了当前正在运行的任务控制块，并用一个附加的`idle_task_cx`来存储进程的上下文信息，进程上下文信息的设置和上一个`Lab`中设置的是一样的，存储了`ra`，`sp`，12个`s`寄存器的信息。

利用`exclusive_access()`解包获得内部的可变引用之后，程序利用`fetch_task`函数将预备队列中的其中一个进程丢出来。
```rust
/// Take a process out of the ready queue
pub fn fetch_task() -> Option<Arc<TaskControlBlock>> {
    //trace!("kernel: TaskManager::fetch_task");
    TASK_MANAGER.exclusive_access().fetch()
}
```
解开`TASK_MANAGER`的封装，得到内部可变引用，利用`pop_front`来获取第一个进程。因此上面的流程就很清楚了，从`TASK_MANAGER`之中弹出一个任务，交给`PROCESSOR`去运行，运行时要调用`__switch`函数，后续流程与`Lab4`是一致的。
### TASK相关的初始化环节
在我们上面的分析中弱化了对于`TaskControlBlock`的初始化，在这边我们还是简单地看一下吧：特别注意，此处的代码复用了上一章的代码，我们重点来查看那些不太一样的地方。
```rust
/// Create a new process
///
/// At present, it is only used for the creation of initproc
pub fn new(elf_data: &[u8]) -> Self {
    // memory_set with elf program headers/trampoline/trap context/user stack
    // 此处的from_elf函数与上一个实验所用的是相同的。
    let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
    let trap_cx_ppn = memory_set
        .translate(VirtAddr::from(TRAP_CONTEXT_BASE).into())
        .unwrap()
        .ppn();
    // alloc a pid and a kernel stack in kernel space
    // 新增关键点1：pid与kernel stack分配
    let pid_handle = pid_alloc();
    let kernel_stack = kstack_alloc();
    let kernel_stack_top = kernel_stack.get_top();
    // push a task context which goes to trap_return to the top of kernel stack
    // 后面的部分没有什么区别，要注意新增加的三个载荷就好了：parent,children,exit_code.
    let task_control_block = Self {
        pid: pid_handle,
        kernel_stack,
        inner: unsafe {
            UPSafeCell::new(TaskControlBlockInner {
                trap_cx_ppn,
                base_size: user_sp,
                task_cx: TaskContext::goto_trap_return(kernel_stack_top),
                task_status: TaskStatus::Ready,
                memory_set,
                parent: None,
                children: Vec::new(),
                exit_code: 0,
                heap_bottom: user_sp,
                program_brk: user_sp,
            })
        },
    };
    // prepare TrapContext in user space
    let trap_cx = task_control_block.inner_exclusive_access().get_trap_cx();
    *trap_cx = TrapContext::app_init_context(
        entry_point,
        user_sp,
        KERNEL_SPACE.exclusive_access().token(),
        kernel_stack_top,
        trap_handler as usize,
    );
    task_control_block
}
```
#### 关键点1：pid与kernel_stack分配
```rust
/// Abstract structure of PID
pub struct PidHandle(pub usize);

impl Drop for PidHandle {
    fn drop(&mut self) {
        //println!("drop pid {}", self.0);
        PID_ALLOCATOR.exclusive_access().dealloc(self.0);
    }
}

/// Allocate a new PID
pub fn pid_alloc() -> PidHandle {
    PidHandle(PID_ALLOCATOR.exclusive_access().alloc())
}
```
在`PID_ALLOCATOR`与`KSTACK_ALLOCATOR`的分配中，我们可以看到其调用了`RecycleAllocator`这一类型。
```rust
lazy_static! {
    static ref PID_ALLOCATOR: UPSafeCell<RecycleAllocator> =
        unsafe { UPSafeCell::new(RecycleAllocator::new()) };
    static ref KSTACK_ALLOCATOR: UPSafeCell<RecycleAllocator> =
        unsafe { UPSafeCell::new(RecycleAllocator::new()) };
}
```
这一数据类型如下所示：采用了一个`recycled`的向量来做这件事情，分配时让`current`往后挪动一位并返回当前分配的`current`值，释放时则检查`id`是否比`current`要小，并且不在`recycled`之中。
```rust
pub struct RecycleAllocator {
    current: usize,
    recycled: Vec<usize>,
}

impl RecycleAllocator {
    pub fn new() -> Self {
        RecycleAllocator {
            current: 0,
            recycled: Vec::new(),
        }
    }
    pub fn alloc(&mut self) -> usize {
        if let Some(id) = self.recycled.pop() {
            id
        } else {
            self.current += 1;
            self.current - 1
        }
    }
    pub fn dealloc(&mut self, id: usize) {
        assert!(id < self.current);
        assert!(
            !self.recycled.iter().any(|i| *i == id),
            "id {} has been deallocated!",
            id
        );
        self.recycled.push(id);
    }
}
```
因此，本质上`pid`的分配还是比较慵懒的，取决于我现在`current`到底跑到了哪里。

现在我们来看`kernelstack`的分配，对于`id`的分配和`pid`一样是相对来说比较随意的，也会根据应用程序的`app_id`决定内核栈的分配位置，然后就会以`FRAMDED`的方式插入到内核地址空间中去。
```rust
/// Return (bottom, top) of a kernel stack in kernel space.
pub fn kernel_stack_position(app_id: usize) -> (usize, usize) {
    let top = TRAMPOLINE - app_id * (KERNEL_STACK_SIZE + PAGE_SIZE);
    let bottom = top - KERNEL_STACK_SIZE;
    (bottom, top)
}
  
/// Kernel stack for a process(task)
pub struct KernelStack(pub usize);

/// allocate a new kernel stack
pub fn kstack_alloc() -> KernelStack {
    let kstack_id = KSTACK_ALLOCATOR.exclusive_access().alloc();
    let (kstack_bottom, kstack_top) = kernel_stack_position(kstack_id);
    KERNEL_SPACE.exclusive_access().insert_framed_area(
        kstack_bottom.into(),
        kstack_top.into(),
        MapPermission::R | MapPermission::W,
    );
    KernelStack(kstack_id)
}
```
到此为止，主题的框架代码分析已经结束了，剩下的是一些系统调用的事情，不过就`OS`这边的流程来看，不是很方便直接看出一些端倪来。
### 进程管理机制的设计与实现
主要`flags`盘点：
1. 初始进程的创建
2. 进程调度机制
3. 进程生成机制
4. 进程资源回收
5. 字符输入机制

1我们自己已经分析过了，2的话注意一下`schedule`函数就差不多了。

3的难点在于`fork`需要让子进程创建一个和父进程几乎完全相同的应用地址空间，`exec`需要在进程控制块层面做一些修改，以让进程能够加载一个新应用的ELF可执行文件中的代码和数据。

4的难点