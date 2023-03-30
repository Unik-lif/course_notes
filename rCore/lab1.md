## 第一章：基本执行环境
### 第一条指令（基础篇）
#### Qemu的启动流程
Qemu的文档其实也是一个大坑，要谨慎地查阅和学习。在Qemu开始执行任何指令之前，首先把两个文件加载到Qemu的物理内存中。

首先，用于当做BootLoader的文件rustsbi-qemu.bin要撞到物理地址为`0x80000000`开头的区域上，同时要把镜像`os.bin`加载到物理地址为`0x80200000`的位置上。

这么装载的原因处于Qemu本身的启动流程：
1. Qemu内固化的小段汇编负责。Qemu的CPU先会初始化为0x1000，因为Qemu的第一条指令位于这个地址中。执行完这一部分小指令后，它会跳转到物理地址`0x80000000`，进入第二个阶段。
2. Bootloader负责。因此，第二阶段的rustsbi-qemu.bin文件会被放在这个位置上。Bootloader将会对计算机进行一些初始化的工作，并且跳转到下一阶段软件的入口，在Qemu上即可实现将计算机控制权移交给镜像`os.bin`的工作。特别的，**对于不同的 bootloader 而言，下一阶段软件的入口不一定相同，而且获取这一信息的方式和时间点也不同：入口地址可能是一个预先约定好的固定的值，也有可能是在 bootloader 运行期间才动态获取到的值。** RustSBI的行为则是固定下一阶段入口地址为`0x80200000`。
3. 内核镜像负责。

真实机器的流程：
1. 在ROM上将后续的BootLoader代码等等载入到物理内存中去，之后再跳转到适当的位置将控制权转移给BootLoader。
2. BootLoader完成一些CPU的初始化工作，将操作系统的镜像从硬盘加载到物理内存中，跳转到适当的位置后，把控制权提供给内核镜像。
3. 系统镜像开工。

#### 链接器
在模块化编程的时候，每个模块都会提供给其他模块公开的全局变量等信息供他们访问。这些信息被分为外部符号和内部符号。内部符号在转化为`.o`文件后，就已经确定下来地址了，但是外部符号则还没有明确。为此，需要利用符号表存储这些外部符号以确定地址。

出于重定位的原因，即便内部符号在目标文件内已经有了一个地址，其也需要被存储在符号表之中。

### 第一条指令（实践篇）
以内联形式`include_str！`写进去的汇编文件，其中的注释是不可以有的，很自然地会报错。

#### linker.ld
在这边通过查看源码可以大体上了解排布的方式，虽然其中的原理奥秘不能完全搞清楚，但也足以窥其一斑。

#### 加载
Qemu不支持动态链接，加载时的原理也较为简单，因此偶尔需要丢弃掉元数据，才能让其较好地工作。

#### 启动时的初始代码：
简要的查看一下pc处往下的十条指令长什么样。
```
(gdb) x/10i $pc
=> 0x1000:  auipc   t0,0x0   ;; t0: 0x00000 000 + PC = 0x00001 000
0x1004:     addi    a1,t0,32 ;; a1: 0x00001 020
0x1008:     csrr    a0,mhartid ;; 从mhartid寄存器中读取hardware thread id到a0
0x100c:     ld      t0,24(t0) ;; load sth from address 0x00001 018, store in t0. we get 0x8000 0000。一个地址一个byte，正好得到0x8000 0000
0x1010:     jr      t0 ;; jump to 0x8000
0x1014:     unimp
0x1016:     unimp
0x1018:     unimp
0x101a:     0x8000
0x101c:     unimp
```
#### 为内核支持函数调用 Q & A
看这一节之前先小测一手，感谢UCB的61c课程给我打了不错的基础：
1. 如何使得函数返回时能够跳转到调用该函数的下一条指令，即使该函数在代码中的多个位置被调用？
- 在栈上存好return address就可以了。
2. 对于一个函数而言，保证它调用某个子函数之前，以及该子函数返回到它之后（某些）通用寄存器的值保持不变有何意义？
- 还原场景。
3. 调用者函数和被调用者函数如何合作保证调用子函数前后寄存器内容保持不变？调用者保存和被调用者保存寄存器的保存与恢复各自由谁负责？它们暂时被保存在什么位置？它们于何时被保存和恢复（如函数的开场白/退场白）？
- 这里存在着callee和caller的calling convention，如果要实现保存和恢复的话，需要在栈上多存存。
4. 在 RISC-V 架构上，调用者保存和被调用者保存寄存器如何划分的？
- t类寄存器可以随便用，s不可以。
5. sp 和 ra 是调用者还是被调用者保存寄存器，为什么这样约定？
- ra需要被调用者保存返回，sp也需要被调用者寄存器好好保存。
6. 如何使用寄存器传递函数调用的参数和返回值？如果寄存器数量不够用了，如何传递函数调用的参数？
- 使用好调用的规范就行了，如果寄存器数量不够，你就存在栈上呗。

#### 实验与练习
1. 实现一个Linux应用程序A，显示当前目录下的文件名：
```Rust
// RTFM is a very good way to learn sth well.
use std::fs;
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() == 2 {
        for file in fs::read_dir(format!("{}", args[1])).unwrap() {
            println!("{}", file.unwrap().path().display());
        }
    } else {
        println!("Error!");
    }
}
```
2. 调用栈：在DEET项目中写过。
3. 实现一个基于rcore/ucore tutorial的应用程序C，用sleep系统调用睡眠5秒（in rcore/ucore tutorial v3: Branch ch1）
```
不会，就是不会
```

qemu的启动位置：似乎是通过reset_vec直接用数组写进去的。
```C
uint32_t reset_vec[10] = {
        0x00000297,                  /* 1:  auipc  t0, %pcrel_hi(fw_dyn) */
        0x02828613,                  /*     addi   a2, t0, %pcrel_lo(1b) */
        0xf1402573,                  /*     csrr   a0, mhartid  */
        0,
        0,
        0x00028067,                  /*     jr     t0 */
        start_addr,                  /* start: .dword */
        start_addr_hi32,
        fdt_load_addr,               /* fdt_laddr: .dword */
        fdt_load_addr_hi32,
                                     /* fw_dyn: */
    };
```

SBI的作用，介于firmware和os之间的一层抽象级，主要的功能是提高可移植性和易用性。

完成了Lab1，实现了彩色输出和print。但是代码很丑很丑（还真是），直接添加了`logging.rs`声明宏定义来制作。丑的不在话下。
### 一个相对优雅的实现：
参考`CH1`中的代码，我们在注释之中简单解读之：
```rust
/*！
本模块利用 log crate 为你提供了日志功能，使用方式见 main.rs.
*/


use log::{self, Level, LevelFilter, Log, Metadata, Record};

struct SimpleLogger; // 面向对象的思想，SimpleLogger作为最终的处理者。

// 为SimpleLogger添加Log特性，Log特性则包含三个函数的简单实现。
// 1. enabled: 判定某一类型是否支持通过log输出
// 2. log: 决定某一类型以何种方式输出
// 3. flush: 刷新缓存中的Records内容
// ----------------------------------------------
// 在有Level和LevelFilter机制的时候，允许采用有优先级的log通知方式。
impl Log for SimpleLogger {
    fn enabled(&self, _metadata: &Metadata) -> bool {
        true
    }
    fn log(&self, record: &Record) {
        if !self.enabled(record.metadata()) {
            return;
        }
        // Level()其实和Record.metadata().level()是一样的。
        let color = match record.level() {
            Level::Error => 31, // Red
            Level::Warn => 93,  // BrightYellow
            Level::Info => 34,  // Blue
            Level::Debug => 32, // Green
            Level::Trace => 90, // BrightBlack
        };
        println!(
            "\u{1B}[{}m[{:>5}] {}\u{1B}[0m",
            color,
            record.level(),
            record.args(), // 经典println所用的参数打印封装方式
        );
    }
    fn flush(&self) {}
}

pub fn init() {
    static LOGGER: SimpleLogger = SimpleLogger; // 第一个SimpleLogger是先前设置的struct，第二个SimpleLogger是后面声明的类型。
    log::set_logger(&LOGGER).unwrap();
    log::set_max_level(match option_env!("LOG") { // 根据输入的参数，即LOG=X，来决定选用的LevelFilter权限级。
        Some("ERROR") => LevelFilter::Error,
        Some("WARN") => LevelFilter::Warn,
        Some("INFO") => LevelFilter::Info,
        Some("DEBUG") => LevelFilter::Debug,
        Some("TRACE") => LevelFilter::Trace,
        _ => LevelFilter::Off,
    });
}

// example in rust docs:
struct SimpleLogger;

impl log::Log for SimpleLogger {
   fn enabled(&self, metadata: &log::Metadata) -> bool {
       true
   }

   fn log(&self, record: &log::Record) {
       if !self.enabled(record.metadata()) {
           return;
       }

       println!("{}:{} -- {}",
                record.level(),
                record.target(),
                record.args());
   }
   fn flush(&self) {}
}
```

### 复盘：
对于linker部分，尚不够清楚，需要继续阅读框架代码以及示例代码，同时观看代码框架讲解视频。

但看样子这讲的不是很好诶。。似乎是在念tutorial。

不过现在看懂了。
### 吐槽
对于`linker`不是很懂。

希望能够在程序员的自我修养一书中得到答案。

### 补充知识：
应用程序执行环境利用**函数调用**去使用标准库的内容，标准库函数依赖于**系统调用**，系统调用依赖于内核与操作系统提供的接口，而内核与操作系统则基于**指令集**在硬件平台上搭建起来。

Rust编译器利用**目标三元组**来描述一个软件运行的目标平台，利用`verbose`可以实现三元组的查看：
```
rustc --version --verbose
```
终端将会返回`host`，`rust-release`，`LLVM version`三元组信息。`host`表示默认的目标平台是什么。

`.cargo`的作用，调整`cargo`工具在项目目录下的行为。设置其`target = "riscv64gc-unknown-none-elf"`，本质上是一种交叉编译的行为。

对于下面的代码对`linker.ld`中的段的调用，很有可能是链接器本身在C中的一些性质。建议去尝试了解。
```C
extern "C" {
    fn sbss();
    fn ebss();
}
(sbss as usize..ebss as usize).for_each(|a| {
    unsafe { (a as *mut u8).write_volatile(0) }
});
```
chatgpt是怎么说的？
```
链接器（Linker）是一个重要的系统工具，它负责将多个目标文件（Object File）或静态库文件（Static Library）合并成一个可执行文件（Executable File）或共享库文件（Shared Library），并分配好程序在内存中的布局。

链接器通常需要一个链接器脚本文件（Linker Script），用于指定程序在内存中的布局。链接器脚本文件中会定义一些符号，如 sbss 和 ebss，用于表示程序数据段的开始和结束位置，以及 bss 段的开始位置。

在 C 语言中，我们可以通过声明 extern int sbss; 和 extern int ebss; 来引用这些符号，然后通过 &sbss 和 &ebss 取得它们的地址。因为 C 语言本身就支持使用外部符号来进行符号地址的引用和调用，所以我们可以在 C 语言中直接使用这些符号。

类似地，在 Rust 中，我们可以使用 extern "C" 关键字来声明一个外部的 C 函数，从而实现与 C 语言的互操作。这样，在 Rust 代码中就可以直接调用 sbss 和 ebss 符号所代表的函数地址，从而实现对这些内存区域的访问。

总的来说，链接器脚本文件定义的符号可以被多个编程语言共享和使用，只要这些编程语言都支持使用外部符号来进行符号地址的引用和调用。这就是为什么 C 语言和 Rust 代码都可以使用链接器脚本文件中定义的符号的原因。
```
也就是说，这其实是源于C的一些性质。