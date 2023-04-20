## 第一章：基本执行环境
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