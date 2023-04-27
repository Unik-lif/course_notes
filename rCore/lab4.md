## 第四章：地址空间
### 引言：
物理内存是操作系统的重要资源，为了让“争抢”不再成为苦恼，操作系统需要达到动态使用内存的目标。通过隔离应用的物理内存空间保证应用之间的安全性，把“有限”物理内存变成“无限”虚拟内存，这是操作系统的一系列重要目标。

`Lab3`中存在的遗留问题恰是上一章给我造成一定困扰的问题，即为何应用程序也需要知道自身代码段在内存中的布局情况。在这一章似乎该问题能够得到不错的解决。如果不解决这个问题，似乎有下面几个较为棘手的问题：
1. 内核提供的内存访问接口不够透明，因此应用程序需要在了解计算机物理内存空间布局的情况、规划自身需要被加载的地址位置的情况下才能正常工作，这非常反人类。
2. 内核并没有对应用的访存行为做限制，没有添加一些保护措施。每个应用都有计算机系统中整个物理内存的读写权力，会造成很多安全上的麻烦。
3. 目前应用的内存空间在使用前就已经被限定死了，内核不可以动态灵活地分配可用的内存空间。

先驱者：`Atlas Supervisor`操作系统
#### 代码改进方向：
前期工作：
1. 将应用程序的起始地址修改为`0x10000`.
2. 为内核添加连续分配内存的功能.
3. 利用物理页帧为单位分配和回收物理内存.

页表建立：
1. 页表项的数据结构搭建.
2. 多级页表起始页帧与占用的物理页帧记录信息.

分页机制：
1. 如何获取应用的ELF中的有效信息，如何解析他们.
2. 页表切换所带来的一些挑战.

### 动态内存分配：
相较于动态内存分配，静态分配存在的问题源于我们再程序中对变量的声明往往仅能应对一部分需求，不够灵活。

动态内存分配使用的是堆内存空间，应用所依赖的基础系统库会直接通过系统调用来向内核请求增加或者缩减应用地址空间内堆的大小。不过这样做也回有不太好的地方，多次做似乎会造成内存空间的浪费，让无法被应用使用的空闲内存碎片变得比较多。这一部分碎片无论是内部碎片还是外部碎片，对于用户来说均是不可见的。

动态分配的缺点确乎是碎片化和内存分配算法带来的更大性能开销，很有可能成为相关内存使用和释放时的巨大瓶颈。

在这一章同样对`Rust`相关的智能指针做了一系列的梳理：

http://rcore-os.cn/rCore-Tutorial-Book-v3/chapter4/1rust-dynamic-allocation.html#rust-heap-data-structures

### 代码结构解读：
```rust
 11 #[no_mangle]
 10 /// the rust entry-point of os
  9 pub fn rust_main() -> ! {
  8     clear_bss();
  7     kernel_log_info();
  6     mm::init();
  5     println!("[kernel] back to world!");
  4     mm::remap_test();
  3     trap::init();
  2     trap::enable_timer_interrupt();
  1     timer::set_next_trigger();
108     task::run_first_task();
  1     panic!("Unreachable in rust_main!");
  2 }
```
#### 虚拟内存：
虚拟内存的初始化主要分为两个部分，首先对堆进行初始化，其次对`frame_allocator`进行初始化，我们尝试逐一分析之。
```rust
  5 /// initiate heap allocator, frame allocator and kernel space
  4 pub fn init() {
  3     heap_allocator::init_heap();
  2     frame_allocator::init_frame_allocator();
  1     KERNEL_SPACE.exclusive_access().activate();
28  }
```
##### 堆初始化：
在堆初始化中，我们看到下面的代码结构。
```rust
 17 #[global_allocator]
 16 /// heap allocator instance
 15 static HEAP_ALLOCATOR: LockedHeap = LockedHeap::empty();
 14
 13 #[alloc_error_handler]
 12 /// panic when heap allocation error occurs
 11 pub fn handle_alloc_error(layout: core::alloc::Layout) -> ! {
 10     panic!("Heap allocation error, layout = {:?}", layout);
  9 }
  8 /// heap space ([u8; KERNEL_HEAP_SIZE])
  7 static mut HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];
  6 /// initiate heap allocator
  5 pub fn init_heap() {
  4     unsafe {
  3         HEAP_ALLOCATOR
  2             .lock()
  1             .init(HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE);
22      }
  1 }
```
与`Lab3`一样利用伙伴系统分配了一个大小为`KERNEL_HEAP_SIZE`的内存空间，在获取锁之后利用`init`对其进行操作，该操作其实是一个范式，能在相关的文档中找到。
##### 栈帧分配与初始化：
该函数的大体框架如图所示：
```rust
 10 type FrameAllocatorImpl = StackFrameAllocator;
  9
  8 lazy_static! {
  7     /// frame allocator instance through lazy_static!
  6     pub static ref FRAME_ALLOCATOR: UPSafeCell<FrameAllocatorImpl> =
  5         unsafe { UPSafeCell::new(FrameAllocatorImpl::new()) };
  4 }
  3 /// initiate the frame allocator using `ekernel` and `MEMORY_END`
  2 pub fn init_frame_allocator() {
  1     extern "C" {
99          fn ekernel();
  1     }
  2     FRAME_ALLOCATOR.exclusive_access().init(
  3         PhysAddr::from(ekernel as usize).ceil(),
  4         PhysAddr::from(MEMORY_END).floor(),
  5     );
  6 }
```
`ekernel`是内核镜像链接脚本中代表内核结束的位置，通过外部引用作为地址信息在函数`init_frame_allocator`中使用。

其中`FRAME_ALLOCATOR`是利用`UPSafeCell`包装好的内部可变的引用，这一点已经在前一个`Lab`中得到了阐述，而内部的单元则换了一层马甲，其实质如下所示。
```rust
  6 /// an implementation for frame allocator
  5 pub struct StackFrameAllocator {
  4     current: usize,
  3     end: usize,
  2     recycled: Vec<usize>,
  1 }

  2 impl StackFrameAllocator {
  1     pub fn init(&mut self, l: PhysPageNum, r: PhysPageNum) {
55          self.current = l.0;
  1         self.end = r.0;
  2         // trace!("last {} Physical Frames.", self.end - self.current);
  3     }
  4 }

   impl FrameAllocator for StackFrameAllocator {
  7     fn new() -> Self {
  6         Self {
  5             current: 0,
  4             end: 0,
  3             recycled: Vec::new(),
  2         }
  1     }
        // etc.
   }
```
其中的`current`和`end`存储的是左物理页号和右物理页号，猜测作为栈的内存在分配上即便是在物理内存上也是连续的，`recycled`到底是何含义暂且不知。

`MEMORY_END`大小为`0x88000000`，是内存结束的地方。猜测上面的`PhysAddr`根据物理地址和`RV64`中物理页大小去标准化（指按照页去选取）相应的物理起始与结束地址。是时候看一看文档对于相关情况的背景介绍了。

-----------------------
### 地址空间章节
`MMU`是内存管理单元，其可以自动地将虚拟地址进行地址转换变为一个物理地址，即这个应用的数据/指令的物理内存位置。

每个应用的地址空间均存在着一个从虚拟地址到物理地址的映射关系，对于不同的应用来说，这个映射关系可能是不同的，即`MMU`可能会将来自不同两个应用地址空间的相同地址转换成不同的物理地址。

为了做区分，`MMU`需要获取`APP`对应的`Token`值，否则难以将相同的虚拟地址映射到不同的物理地址处。

![](http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/simple-base-bound.png)

演变流程：

Base & Bound -> Segmentation -> Page mechanism.
### SV39多级页表
在阅读之前，我们看看官方手册。
#### 手册内容
`satp`寄存器是一个长度为`SXLEN-bit`的读写寄存器，在`SXLEN-bit`分别为`32`和`64`时，其结构分别如下所示：
```
     31    30        22 21            0
---------------------------------------
|   mode   |  ASID    |    PPN        |
---------------------------------------
for Sv32.

63       60 59       44 43            0
---------------------------------------
|   mode   |  ASID    |    PPN        |
---------------------------------------
for Bare, Sv39, Sv48, Sv57.

Bare -> supervisor virtual address are equal to supervisor physical addresses.

mode: select current address-translation scheme.
ASID: facilitates address-translation fences on a per-address-space basis.
PPN: physical page number.

ASID and the page table base address will be stored in the same CSR to allow the pair to be changed atomically on a context switch.
```
对于rCore中的情况，采用的`Sv39`这种模式，对应的前四个比特应该是这么设置的：`0b1000`。

`Sv39`一共有`39`位是虚拟地址空间，那就说明最大可用的虚拟内存空间是`512GB`，这对于很多系统来说是很充足的。

规定`Sv39`的高位`63~39`比特必须和第`38`位相同（什么`sign-extension`，我笑了）。

```
38        30 29       21 20       12 11              0
------------------------------------------------------
|   VPN[2]  |   VPN[1]  |   VPN[0]  |   page offset  |
------------------------------------------------------


55        30 29       21 20       12 11              0
------------------------------------------------------
|   PPN[2]  |   PPN[1]  |   PPN[0]  |   page offset  |
------------------------------------------------------

```
Page tables contain $2^{9}$ page table entries, eight bytes each.

-----------------------------------------