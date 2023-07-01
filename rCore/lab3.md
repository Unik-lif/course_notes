## 第三章：多道程序与分时多任务
### 引言：
因内存容量增大可以驻留多个应用了，科学家们为了提高处理器的利用率提出了多道程序模型。

在该模型中，与批处理类似，处理器只有在一个程序执行完毕后或主动放弃执行时，处理器才能执行另一个程序。但并不需要和批处理一样必须要一个一个地去加载相关的应用，它们可以一开始均被加载到内存，从而节省了一些时间。

#### 协作式操作系统：
多道程序模型存在自己的弊端，如果一个程序不让出处理器，其他程序是无法执行的。因此，如果在应用执行IO操作或者空闲的时候，可以主动释放处理器，则能一定程度上提高效率，这就是协作式多任务的操作系统。
#### 抢占式：
系统需要有一个强制打断应用程序执行的方法，就和病危的人需要走绿色通道进入急救室一样，否则轻重缓急之事均会耽误。操作系统可利用固定时长为时间间隔的外设中断，如时钟中断来强制打断一个程序的执行，这样程序只能运行一段时间片，就让出处理器，且操作系统可以在处理外设的IO相应后，让不同应用程序分时占用处理器执行。

这便是抢占式分时多任务模型。
### 代码框架解读：
进入`main`函数，我们看到下面的步骤：
```rust
  4 #[no_mangle]
  3 /// the rust entry-point of os
  2 pub fn rust_main() -> ! {
  1     clear_bss();
        // print for every segment.
99      kernel_log_info();
  1     heap_alloc::init_heap();
  2     trap::init();
  3     loader::load_apps();
  4     trap::enable_timer_interrupt();
  5     timer::set_next_trigger();
  6     task::run_first_task();
  7     panic!("Unreachable in rust_main!");
  8 }
```
`kernel_log_info`是打印环节，我们略过。
#### 初始化堆：
首先，利用`heap_alloc::init_heap()`函数对堆进行分配：
```rust
  2 pub fn init_heap() {
  1     unsafe {
16          HEAP_ALLOCATOR // locked version of HEAP.
  1             .lock()
  2             .init(HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE);
  3     }
  4 }
```
利用伙伴系统分配，分配了一个大小为`KERNEL_HEAP_SIZE`，即`0x20000`大小的堆，具体用法参考下面的链接：
https://docs.rs/buddy_system_allocator/latest/buddy_system_allocator/struct.LockedHeap.html#method.empty
#### Trap_init：
其次，利用`trap_init`把中断处理器开起来。代码已经在上一节中进行分析了。

时钟代码似乎是源于`RustSBI`的设置，使得我们身为`S`特权级依旧能够操纵时间。

但是相较于上次的代码，本次还添加了中断处理部分：

我们简单看一下，由于程序分析到这里，本质上还没有进程（task）跑起来，所以只是简单看一下。
```rust
  4         Trap::Interrupt(Interrupt::SupervisorTimer) => {
  5             set_next_trigger();
  6             suspend_current_and_run_next();
  7         }
```
#### 初始化loader，并用它完成应用程序的加载：
第三，我们用`loader`完成多个应用程序的加载。
```rust
  1 /// Load nth user app at
65  /// [APP_BASE_ADDRESS + n * APP_SIZE_LIMIT, APP_BASE_ADDRESS + (n+1) * APP_SIZE_LIMIT).
  1 pub fn load_apps() {
  2     extern "C" {
  3         fn _num_app();
  4     }
  5     let num_app_ptr = _num_app as usize as *const usize; // in link_app.S
  6     let num_app = get_num_app(); // simply read the first element.
  7     let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
  8     // clear i-cache first
  9     unsafe {
 10         asm!("fence.i");
 11     }
 12     // load apps, all at once.
 13     for i in 0..num_app {
 14         let base_i = get_base_i(i);
 15         // clear region
 16         (base_i..base_i + APP_SIZE_LIMIT)
 17             .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
 18         // load app from data section to memory
 19         let src = unsafe {
 20             core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
 21         };
 22         let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
 23         dst.copy_from_slice(src);
 24     }
 25 }
```
不同于批处理需要一个一个地装载应用程序，这里为每个应用程序分配了大小和先前分配的堆所相同大小的空间，即`APP_SIZE_LIMIT`。然后依次把他们这样码上来。
```rust
 12 /// Get base address of app i.
 13 fn get_base_i(app_id: usize) -> usize {
 14     APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
 15 }
```
特别的，在一开始编译相关二进制应用程序时也参考了这样的约定：
```shell
    Finished release [optimized] target(s) in 2.63s
[build.py] application ch2b_bad_address start with address 0x80400000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.93s
[build.py] application ch2b_bad_instructions start with address 0x80420000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.94s
[build.py] application ch2b_bad_register start with address 0x80440000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.92s
[build.py] application ch2b_hello_world start with address 0x80460000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.95s
[build.py] application ch2b_power_3 start with address 0x80480000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.94s
[build.py] application ch2b_power_5 start with address 0x804a0000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.94s
[build.py] application ch2b_power_7 start with address 0x804c0000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.94s
[build.py] application ch3b_yield0 start with address 0x804e0000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 0.94s
[build.py] application ch3b_yield1 start with address 0x80500000
   Compiling user_lib v0.1.0 (/mnt/d/2023s-rcore-Unik-lif/user)
    Finished release [optimized] target(s) in 1.00s
[build.py] application ch3b_yield2 start with address 0x80520000
make[1]: Leaving directory '/mnt/d/2023s-rcore-Unik-lif/user'
```
具体可以在`src`下的脚本`build.py`中查看清楚，其修改了每个应用程序编译时的`linker.ld`代码段所存放的位置，从而方便把相关的`binary`文件代码段加载进来。

我在这边有一个问题，我们不是利用`.incbin`装载的应用程序吗？这只不过是把二进制动态装进来罢了，为啥还要对应用程序的链接器做修改？

为什么要这么严苛？为什么即便是我们利用`.incbin`装载进来的二进制文件也需要在链接的一开始让程序知道自己在哪里运行？为什么要让程序一开始就知道自己的代码应该被装载在哪里？

可以参考下面的链接：
https://nju-projectn.github.io/ics-pa-gitbook/ics2020/4.2.html

简单思考一下，把我自己的思路放在这里写一写：

其实讲的很有道理，如果程序自己本身是绝对地址编写的，不管它之后会装载到那里，运行应用程序这个过程本身，程序会认为自己会按照先前绝对地址编写的方式来运行。然而，真实在加载到操作系统中后，倘若其发现情况并不是这样，自然会发生度量衡不符了。这就和螺丝和螺帽一样，在**自重定位**这一“万能钥匙”添加之前，必须做到严丝合缝，否则会造成应用程序的运行异常。
#### 定时内陷：以让操作系统间隙掌权
##### enable_timer_interrupt与嵌套中断
由RISCV自身体系结构较为优雅地解决了：参考Tutorial
```
当 Trap 发生时，sstatus.sie 会被保存在 sstatus.spie 字段中，同时 sstatus.sie 置零， 这也就在 Trap 处理的过程中屏蔽了所有 S 特权级的中断；

当 Trap 处理完毕 sret 的时候， sstatus.sie 会恢复到 sstatus.spie 内的值。
```
不过这个过程我们目前似乎还没看到，看看之后能不能追溯一下。

到目前为止，如果系统正在内陷过程中，我们会屏蔽内陷。
```rust
  4 /// enable timer interrupt in supervisor mode
  3 pub fn enable_timer_interrupt() {
  2     unsafe {
  1         sie::set_stimer();
43      }
  1 }
```
`sie::set_stimer`会让`mie`寄存器的`STIE`位设置为1，以开启内核态的时钟中断。
##### 时钟：set_next_trigger
`set_next_trigger()`函数会设置下一次内陷的时间，具体时间是`CLOCK_FREQ/TICKS_PER_SEC`之后。
```rust
  8 /// get current time in microseconds
  7 #[allow(dead_code)]
  6 pub fn get_time_us() -> usize {
  5     time::read() * MICRO_PER_SEC / CLOCK_FREQ
  4 }
```
既然这个时间是这么算的，`time::read() / CLOCK_FREQ`反映的应该就是秒。`time::read()`告知一共有多少个时间增量，而CLOCK_FREQ则反映每秒有多少个时间增量。

10ms 之内计数器的增量: `CLOCK_FREQ / TICKS_PER_SEC`. `TICKS_PER_SEC`其实表示的是一秒钟内，内陷发生的次数，并不是上面我们说的那个记录。

坦白说，这个常数名字`TICKS_PER_SEC`没有取好，很容易让人混淆（尤其是同时在看注释的情况下，个人认为`get_time`的注释不妥）。
```rust
15  /// Get the current time in ticks
  1 pub fn get_time() -> usize {
  2     time::read()
  3 }
```
异常将会在trap handler里体现：
```rust
67          Trap::Interrupt(Interrupt::SupervisorTimer) => {
  1             set_next_trigger();
  2             suspend_current_and_run_next();
  3         }
```
#### 运行程序：run_first_task
在做好了前述的准备后，我们开始运行相关的应用程序。
```rust
  1 /// Run the first task in task list.
141 pub fn run_first_task() {
  1     TASK_MANAGER.run_first_task();
  2 }
```
`TASK_MANAGER`用于管理相关的`Task`，与lab2的`Manager`类似，其是`Task Manager`的实例化。但`TASK_MANAGER`的管理方式已经很接近Linux中`TASK_STRUCT`链表中的管理方式了。
```rust
 19 /// The task manager, where all the tasks are managed.
 18 ///
 17 /// Functions implemented on `TaskManager` deals with all task state transitions
 16 /// and task context switching. For convenience, you can find wrappers around it
 15 /// in the module level.
 14 ///
 13 /// Most of `TaskManager` are hidden behind the field `inner`, to defer
 12 /// borrowing checks to runtime. You can see examples on how to use `inner` in
 11 /// existing functions on `TaskManager`.
 10 pub struct TaskManager {
  9     /// total number of tasks
  8     num_app: usize, // MAX_NUM: 16.
  7     /// use inner value to get mutable access
  6     inner: UPSafeCell<TaskManagerInner>, // We've discussed before.
  5 }
  4
  3 /// Inner of Task Manager
  2 pub struct TaskManagerInner {
  1     /// task list
45      tasks: [TaskControlBlock; MAX_APP_NUM],
  1     /// id of current `Running` task
  2     current_task: usize,
  3 }
```
`TCB`块的维护在`task.rs`中得到了体现：其中主要储存`Task`的状态和相关的上下文信息。
```rust
  8 //! Types related to task management
  7
  6 use super::TaskContext;
  5
  4 /// The task control block (TCB) of a task.
  3 #[derive(Copy, Clone)]
  2 pub struct TaskControlBlock {
  1     /// The task status in it's lifecycle
9       pub task_status: TaskStatus,
  1     /// The task context
  2     pub task_cx: TaskContext,
  3 }
  4
  5 /// The status of a task
    // 看样子有四种状态，这很OSTEP
  6 #[derive(Copy, Clone, PartialEq)]
  7 pub enum TaskStatus {
  8     /// uninitialized
  9     UnInit,
 10     /// ready to run
 11     Ready,
 12     /// running
 13     Running,
 14     /// exited
 15     Exited,
 16 }
```
`TASK_MANAGER`的创立：利用`lazy_static!`宏，使得操作仅在被调用时才真正分配内存空间给它。
```rust
 20 lazy_static! {
 19     /// Global variable: TASK_MANAGER
 18     pub static ref TASK_MANAGER: TaskManager = {
 17         let num_app = get_num_app();
 16         let mut tasks = [TaskControlBlock {
 15             task_cx: TaskContext::zero_init(),
 14             task_status: TaskStatus::UnInit,
 13         }; MAX_APP_NUM];
 12         for (i, task) in tasks.iter_mut().enumerate() {
 11             task.task_cx = TaskContext::goto_restore(init_app_cx(i));
 10             task.task_status = TaskStatus::Ready;
  9         }
  8         TaskManager {
  7             num_app,
  6             inner: unsafe {
  5                 UPSafeCell::new(TaskManagerInner {
  4                     tasks,
  3                     current_task: 0,
  2                 })
  1             },
70          }
  1     };
  2 }
```
其中的函数`goto_restore`比较重要。选取__restore作为返回地址，设置`init_app_cx(i)`的返回值作为栈地址，`init_app_cx(i)`函数返回的值是受应用程序装载时的`id`所影响的`KERNEL_STACK`内经过`push`应用操作后的栈地址。
```rust
  3     /// Create a new task context with a trap return addr and a kernel stack pointer
  2     pub fn goto_restore(kstack_ptr: usize) -> Self {
  1         extern "C" {
27              fn __restore();
  1         }
  2         Self {
  3             ra: __restore as usize,
  4             sp: kstack_ptr,
  5             s: [0; 12],
  6         }
  7     }

    2 /// get app info with entry and sp and save `TrapContext` in kernel stack
  1 pub fn init_app_cx(app_id: usize) -> usize {
94      KERNEL_STACK[app_id].push_context(TrapContext::app_init_context(
  1         get_base_i(app_id),
  2         USER_STACK[app_id].get_sp(),
  3     )) // initialize the user cx of app. Set the entry and the stack pointer.
  4 }
```
对该实例化的`TASK_MANAGER`的执行函数为`run_first_task`：
```rust
  1     fn run_first_task(&self) -> ! {
80          let mut inner = self.inner.exclusive_access(); // same time only one. See lab2 for more info.
  1         let task0 = &mut inner.tasks[0];
            // all ready, set to runnning here.
  2         task0.task_status = TaskStatus::Running;
            // We only set the status of the task0, but task0 is not running now. 
  3         let next_task_cx_ptr = &task0.task_cx as *const TaskContext;
  4         drop(inner);
  5         let mut _unused = TaskContext::zero_init();
  6         // before this, we should drop local variables that must be dropped manually
            // __switch: key function for task scheduling.
            // jump to next_task_cx_ptr, we run the first task here.
            // ret to ra in next_task_cx_ptr.
  7         unsafe {
  8             __switch(&mut _unused as *mut TaskContext, next_task_cx_ptr);
  9         }
 10         panic!("unreachable in run_first_task!");
 11     }
```
`switch`代码会切换应用的上下文环境，并用`ret`返回到预先`task`在`ra`寄存器内设置好的位置`__restore`代码处（见`lazy_static!`位置）。

从`__restore`处通过`sret`返回，然后执行完退出后走`syscall`给捕获到，和`LAB2`后续处理流程一样，完事儿了。
### 其他需要解答的问题：
但是还有个功能我们没有细说，交给明天吧阿巴巴打夺夺。

剩下的部分请参考文档：http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter3/3multiprogramming.html#sys-yield-sys-exit ，此处有较为详细的描述，我就不赘述了。

目光主要聚焦在函数`exit_current_and_run_next`和`suspend_current_and_run_next`上，随意跟踪之即可。

每个`APP`都拥有一个内核栈。
```rust
 14 static KERNEL_STACK: [KernelStack; MAX_APP_NUM] = [KernelStack {
 15     data: [0; KERNEL_STACK_SIZE],
 16 }; MAX_APP_NUM];
 17
 18 static USER_STACK: [UserStack; MAX_APP_NUM] = [UserStack {
 19     data: [0; USER_STACK_SIZE],
 20 }; MAX_APP_NUM];
```

此处的`next`源于一开始声明时的`tasks`数组的排布：
```rust
  6     fn run_next_task(&self) {
  5         if let Some(next) = self.find_next_task() {
  4             let mut inner = self.inner.exclusive_access();
  3             let current = inner.current_task;
  2             inner.tasks[next].task_status = TaskStatus::Running;
  1             inner.current_task = next; // note here!
126             let current_task_cx_ptr = &mut inner.tasks[current].task_cx as *mut TaskContext;
  1             let next_task_cx_ptr = &inner.tasks[next].task_cx as *const TaskContext;
  2             drop(inner);
  3             // before this, we should drop local variables that must be dropped manually
  4             unsafe {
  5                 __switch(current_task_cx_ptr, next_task_cx_ptr);
  6             }
  7             // go back to user mode
  8         } else {
  9             panic!("All applications completed!");
 10         }
 11     }
```

### 实验实现思路：
拓展全局变量`TASK_MANAGER`的接口，以使得其能够返回必要的信息以填充`TASK_INFO`信息。
```rust
 24     /// accumulate syscall_times due to sycall number i.
 23     /// Will be called in src/syscall/process.rs
 22     fn add_one_syscall(&self, sys_num: usize) {
 21         let mut inner = self.inner.exclusive_access();
 20         let current = inner.current_task;
 19         inner.tasks[current].taskinfo.syscall_times[sys_num] += 1;
 18     }
 17
 16     /// Pass the pointer of the current task info.
 15     fn pass_syscall_info(&self) -> SyscallInfo {
 14         let inner = self.inner.exclusive_access();
 13         let current = inner.current_task;
 12         inner.tasks[current].taskinfo
 11     }
 10
  9     /// Pass the status of the current task.
  8     fn pass_task_status(&self) -> TaskStatus {
  7         let inner = self.inner.exclusive_access();
  6         let current = inner.current_task;
  5         inner.tasks[current].task_status
  4     }
```
函数`add_one_syscall`将会放置在系统调用之前，以在相应的桶中进行系统调用数目的统计。

延拓`TCB`控制块，以使得其能够存储系统调用信息和相关的时间。
```rust
 10 #[derive(Copy, Clone)]
  9 pub struct TaskControlBlock {
  8     /// The task status in it's lifecycle
  7     pub task_status: TaskStatus,
  6     /// The task context
  5     pub task_cx: TaskContext,
  4     /// Lab1: The task info
  3     pub taskinfo: SyscallInfo,
  2 }

  8 /// Lab1: my taskinfo.
  7 /// The syscall info of a task.
  6 #[derive(Copy, Clone)]
  5 pub struct SyscallInfo {
  4     /// The numbers of syscall called by task
  3     pub syscall_times: [u32; MAX_SYSCALL_NUM],
  2     /// Total running time of a task.
  1     pub time: usize,
38  }
```
`rust.vim`和`systansitic`插件可以让编译过得很轻松，这里爆赞！

提供了这些接口后，任务就很容易了。

完事了。吐槽一点，如果不叫报告，你过不了`ci-test`，太好笑了。