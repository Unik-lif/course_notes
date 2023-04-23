## 第二章：基本执行环境
### 引言：
批处理的诞生背景：管理无需或者仅需少量用户交互就可运行的程序。因为计算机运行是很珍贵的，就当是煮火锅要花很多电费吧，我要积攒一部分好菜再去涮火锅，这样才比较合适。

本章的主线任务：

用户态的分割 -> 系统调用的实现 -> 轮转启动运行的操作系统 -> 资源调度/trap上下文的保存与恢复。
### 特权级机制：
其实就是画饼啊哈哈，比如，一个合理的操作系统应该是什么样的？

rCore作者们似乎是认为，特权级的出现源于多任务的调度使得作为内核的部件的安全性受到了compromise。

因此，一个比较贴心的应用程序应该做到两方面的限制：
1. 不能访问任意的地址空间
2. 不能执行某些可能破坏计算机系统的指令

用户采用传统的函数调用方式，如`call`与`ret`将会直接绕过硬件的特权级保护检查。所以可以设计新的机器指令，分别是`ecall`与`eret`。

#### RISCV特权级架构
RISCV中定义了四种特权级。

| 级别 | 编码 | 名称 |
|  ----  | ----  | --- |
|  0 | 00 | 用户模式 user |
| 1 | 01 | 监督模式 supervisor|
| 2 | 10 | 虚拟监督模式 hypervisor|
| 3 | 11| 机器模式 machine|

rCore的操作系统会使用M/S/U共三个特权级，之间分别利用SBI和ABI进行交互。

RISCV存在部分特权指令，以S模式为例，一般有两类：
1. 本身属于高特权级的指令
2. 访问了高特权级才能访问的内存或者寄存器

### 应用程序实现
实现应用程序的出发点：
1. 应用程序的内存布局
2. 应用程序发出的系统调用如何处理

### 代码解读：
1. base_address被写进了build.py这个脚本中，因此在linker.ld中似乎没有看到0x80400000的值。（对应版本rCore-2023S）
2. 关于APP_MANAGER：参考alpha.1文档
```rust
lazy_static! {
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe { UPSafeCell::new({
        extern "C" { fn _num_app(); }
        let num_app_ptr = _num_app as usize as *const usize;
        let num_app = num_app_ptr.read_volatile();
        let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
        let app_start_raw: &[usize] =  core::slice::from_raw_parts(
            num_app_ptr.add(1), num_app + 1
        );
        app_start[..=num_app].copy_from_slice(app_start_raw);
        AppManager {
            num_app,
            current_app: 0,
            app_start,
        }
    })};
}
```
num_app + 1: 把最后一位停下来的app地址也计量进去。

num_app_ptr.add(1): APP_MANAGER的第一个参数是app总数，从第二个参数开始才是相关的应用程序地址。
3. lazy_static!：对于全局变量的初始化在第一次被使用到的时候才进行，所以lazy这一描述是精准的。
4. asm!("fence.i"): 有多个应用程序在运行，为了防止不一致的情况，需要对指令缓存进行清空，设置该指令即完成一个屏障作用。
### 特权级切换：
批处理操作系统应用程序执行环境：
1. 启动应用程序的时候，初始化应用程序的用户态上下文，并切换到用户态。
2. 应用程序发起系统调用，让批处理操作系统来处理。
3. 应用程序执行出错，则杀死，加载运行下一个应用。
4. 执行结束后，从操作系统中加载下一个应用。

我们在编写运行在 S 特权级的批处理操作系统中的 Trap 处理相关代码的时候，就需要使用如下所示的 S 模式的 CSR 寄存器。

进入 S 特权级 Trap 的相关 CSR

| CSR 名 | 该 CSR 与 Trap 相关的功能 |
| - | - |
| sstatus | SPP 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息 |
| sepc | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址 |
| scause | 描述 Trap 的原因 |
| stval | 给出 Trap 附加信息 |
| stvec | 控制 Trap 处理代码的入口地址|

参考lec1.md后续对于手册的整理即可理解。
### 用户栈和内核栈
同时对内存栈和用户栈做初始化。
```rust
// os/src/batch.rs

const USER_STACK_SIZE: usize = 4096 * 2;
const KERNEL_STACK_SIZE: usize = 4096 * 2;

#[repr(align(4096))]
struct KernelStack {
    data: [u8; KERNEL_STACK_SIZE],
}

#[repr(align(4096))]
struct UserStack {
    data: [u8; USER_STACK_SIZE],
}

static KERNEL_STACK: KernelStack = KernelStack { data: [0; KERNEL_STACK_SIZE] };
static USER_STACK: UserStack = UserStack { data: [0; USER_STACK_SIZE] };

// for UserStack. + USER_STACK_SIZE -> find the bottom of the Stack.
impl UserStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}
```
上面这个流程总体上还是比较清楚的。
### Trap上下文
Trap context size:
```rust
// os/src/trap/context.rs
// size: (32 + 2) * 8.
#[repr(C)]
pub struct TrapContext {
    pub x: [usize; 32],
    pub sstatus: Sstatus,
    pub sepc: usize,
}
```

```r
# os/src/trap/trap.S

.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm

.align 2 # align for 2^{N}, N is 2.
__alltraps:
    csrrw sp, sscratch, sp # sscratch: hold supervisor context pointer. 
    # 1, 2, 3: sscratch fetch 3, the user stack.
    # the sp now fetch sscratch, the kernel stack.
    # now sp->kernel stack, sscratch->user stack

    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8
    # save general-purpose registers
    sd x1, 1*8(sp)
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp)
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus # store current privilege to t0.
    csrr t1, sepc # store return address to t1.
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # after above instructions, the context is stored, and able to switch now. 

    # read user stack from sscratch and save it on the kernel stack
    # store sscratch, the user stack, into t2.
    # store use stack into ths 2*8(sp) position.
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp
    call trap_handler
```
说白了，在内核栈里存放用户层次的上下文环境。存储完后，我们再把sp塞进来给a0，准备调用trap_handler。

注意到trap_handler的代码长这样：
```rust
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext
```
第一个参数是trapcontext的地址，而sp恰好存着这玩意。于是真相大白。

sscratch寄存器的最大意义：作为通用寄存器之外的一个用于暂时存放内核栈的存在，便于操作系统在内核栈与用户栈之间切换。

之后的部分阅读文档资料即可。

特别的，为了有一个全局的认识，我们最好从main.rs下向下读代码遍历梳理一遍，这样才不会**盲人摸象**。

### Other notes：
一些值得注意的地方：
- mod的遍历顺序：在main.rs中`pub mod trap;`引入了trap这个模块，但并不存在名为`trap.rs`的文件。Rust正确的遍历寻找顺序一般是：
```
Inline, within curly brackets that replace the semicolon following mod garden
In the file src/garden.rs
In the file src/garden/mod.rs
```
因此在我们这边看到`trap/mod.rs`起到了相应的作用。
- trap::init函数

`stvec`寄存器存放的是trap处理代码的入口地址。
```rust
  5 pub fn init() {
  4     extern "C" {
  3         fn __alltraps();
  2     }
  1     unsafe {
34          stvec::write(__alltraps as usize, TrapMode::Direct);
  1     }
  2 }
```
- 为什么就这点代码就足够处理中断和特权级的问题了？
```
你先别急，具体的事情我们可以先进入trap handler再尝试去做。

sstatus 的 SPP 字段会被修改为 CPU 当前的特权级（U/S）。

sepc 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址。

scause/stval 分别会被修改成这次 Trap 的原因以及相关的附加信息。

CPU 会跳转到 stvec 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从Trap 处理入口地址处开始执行。
```
- __alltrap干了什么事情？
1. 保存上下文环境，即保存用户栈的全部相关寄存器信息，切换到内核栈，把之前的用户栈信息一股脑儿先放在内核栈中，记录sstatus（特权级信息），sepc（返回地址PC值）
```assembly
  3     # we can use t0/t1/t2 freely, because they were saved on kernel stack
  4     csrr t0, sstatus
  5     csrr t1, sepc
  6     sd t0, 32*8(sp)
  7     sd t1, 33*8(sp)
```
在这边需要把sstatus和sepc先储存在临时寄存器t0，t1，因为这一类特权级寄存器不可以直接用来和地址进行数据交换，需要通过通用寄存器来辅助。
2. 以用户栈地址作为参数，送给trap_handler函数来做
- trap_handler做了什么？
1. 读取scause和stval寄存器信息（该信息不会在内核栈和用户栈中记录），确认异常原因。
2. 根据错误原因各自处理之，或调用系统调用，或直接报错并运行下一个应用程序，进入`run_next_app`支线。
3. 返回context
- 之后呢，之后会进入`__restore`函数，这个函数会做什么？
1. 把处理完的`context`返回，刚刚我们再trap_handler让指令跳转到下一步的地址处了，毕竟riscv指令的长度是4。与此同时恢复特权级寄存器的值（存放在内核栈中）
2. 释放内核栈空间，从内核态转到用户态。
- 程序是怎么跑起来的？
```
具体来说这个问题得等到你去看`batch::run_next_app`才会清晰起来，而这一部分源码在rCore有很耐心的阐述，不过组织的方式容易让人陷入到细节和死胡同中。
```

到这里，实验二我感觉基本分析的差不多了，没有什么可以讲的了。

顺带着吐槽一下清华大佬们写代码时的一些奇技淫巧，果然非一日之功哈哈。:)