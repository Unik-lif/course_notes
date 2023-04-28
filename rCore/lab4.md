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
#### 虚拟内存初始化：
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

`MEMORY_END`大小为`0x88000000`，是内存结束的地方。我们通过跑程序，可以发现`ekernel`的大小大概是`0x82273000`。特别的，在我们打印这些信息的时候，我们并没有启动虚拟内存转换器，因此这里反映的值显然是物理地址。

猜测上面的`PhysAddr`根据物理地址和`RV64`中物理页大小去标准化（指按照页去选取）相应的物理起始与结束地址。是时候看一看文档对于相关情况的背景介绍了。

-----------------------
**地址空间章节**

`MMU`是内存管理单元，其可以自动地将虚拟地址进行地址转换变为一个物理地址，即这个应用的数据/指令的物理内存位置。

每个应用的地址空间均存在着一个从虚拟地址到物理地址的映射关系，对于不同的应用来说，这个映射关系可能是不同的，即`MMU`可能会将来自不同两个应用地址空间的相同地址转换成不同的物理地址。

为了做区分，`MMU`需要获取`APP`对应的`Token`值，否则难以将相同的虚拟地址映射到不同的物理地址处。

![](http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/simple-base-bound.png)

演变流程：

Base & Bound -> Segmentation -> Page mechanism.

**SV39多级页表**

在阅读之前，我们看看官方手册。

**手册内容**

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
简单打印先前的一些地址信息如下：
```shell
[TRACE] [kernel] .text [0x80200000, 0x8020d000)
[DEBUG] [kernel] .rodata [0x8020d000, 0x80212000)
[ INFO] [kernel] .data [0x80212000, 0x80262000)
[ WARN] [kernel] boot_stack top=bottom=0x80272000, lower_bound=0x80262000
[ERROR] [kernel] .bss [0x80272000, 0x82273000)
[ INFO] .text [0x80200000, 0x8020d000)
[ INFO] .rodata [0x8020d000, 0x80212000)
[ INFO] .data [0x80212000, 0x80262000)
[ INFO] .bss [0x80262000, 0x82273000)
[ INFO] mapping .text section
[ INFO] mapping .rodata section
[ INFO] mapping .data section
[ INFO] mapping .bss section
[ INFO] mapping physical memory
```
理解了`RISCV`针对`Sv39`的相关内存布置后，这一部分源码理解起来很容易，我们找找源代码中比较有趣的例子做分析：
```rust
 22 impl From<VirtAddr> for usize {
 21     fn from(v: VirtAddr) -> Self {
 20         if v.0 >= (1 << (VA_WIDTH_SV39 - 1)) {
 19             v.0 | (!((1 << VA_WIDTH_SV39) - 1))
 18         } else {
 17             v.0
 16         }
 15     }
 14 }
```
这部分代码的核心理念，其实在于把`usize`转化成虚拟地址时针对第`38`位在`Sv39`中的特殊意义做了一些加工（`63 rd ~ 39 th bit` equals to `38 th bit`）。

下面这部分代码说白了就是根据`4 KB`的页大小考虑选取页有关`floor, ceil`的事宜，没什么好讲的。
```rust
 26 /// virtual address impl
 25 impl VirtAddr {
 24     /// Get the (floor) virtual page number
 23     pub fn floor(&self) -> VirtPageNum {
 22         VirtPageNum(self.0 / PAGE_SIZE)
 21     }
 20
 19     /// Get the (ceil) virtual page number
 18     pub fn ceil(&self) -> VirtPageNum {
 17         VirtPageNum((self.0 - 1 + PAGE_SIZE) / PAGE_SIZE)
 16     }
 15
 14     /// Get the page offset of virtual address
 13     pub fn page_offset(&self) -> usize {
 12         self.0 & (PAGE_SIZE - 1)
 11     }
 10
  9     /// Check if the virtual address is aligned by page size
  8     pub fn aligned(&self) -> bool {
  7         self.page_offset() == 0
  6     }
  5 }
```
----------------------------
我们继续分析原来的代码，因此可以推测下面这一串代码是以`ekernel`和`memory_end`地址作为物理页的最大范围，之后的虚拟内存和栈帧分配应该会是在这样一个物理页范围内执行相应的操作。
```rust
  2     FRAME_ALLOCATOR.exclusive_access().init(
  3         PhysAddr::from(ekernel as usize).ceil(),
  4         PhysAddr::from(MEMORY_END).floor(),
  5     );
```
##### KENREL_SPACE相应的操作
```rust
  5 /// initiate heap allocator, frame allocator and kernel space
  4 pub fn init() {
  3     heap_allocator::init_heap();
  2     frame_allocator::init_frame_allocator();
  1     KERNEL_SPACE.exclusive_access().activate();
28  } 
```
我们完成了堆的初始化，也完成了用于虚拟地址空间的物理页范围的确定工作，下面需要对`KERNEL_SPACE`做启动工作。简单整理一下这里的数据结构，其对应的树形图如下。
```
KERNEL_SPACE -> MemorySet -> page_table: PageTable
                                                  
                          -> areas: Vec<MapArea>

PageTable -> root_ppn: PhysPageNum
          -> frames: Vec<FrameTracker> -> ppn: PhysPageNum

MapArea -> vpn_range: VPNRange 
        -> data_frames: BTreeMap<VirtPageNum, FrameTracker>
        -> map_type: MapType
        -> map_perm: MapPermission
```
`KERNEL_SPACE`是`Memory Set`利用内部可变指针实例化得到的一个全局变量。
```rust
 31 lazy_static! {
 30     /// The kernel's initial memory mapping(kernel address space)
 29     pub static ref KERNEL_SPACE: Arc<UPSafeCell<MemorySet>> =
 28         Arc::new(unsafe { UPSafeCell::new(MemorySet::new_kernel()) });
 27 }
```
对于`MemorySet::new_kernel`这一函数，涉及的内容似乎还挺多的。我们尝试逐步分析。
- 初始化操作：
```rust
82      pub fn new_kernel() -> Self {
            // 利用new_bare建立一个空的、动态分配的MemorySet集合
  1         let mut memory_set = Self::new_bare();
  2         // map trampoline
  3         memory_set.map_trampoline();
```
在这一步中，利用函数`new_bare`建立一个空的集合。
```rust
  6     /// Create a new empty `MemorySet`.
  5     pub fn new_bare() -> Self {
  4         Self {
  3             page_table: PageTable::new(),
  2             areas: Vec::new(),
  1         }
49      }
```
之后，利用`map_trampoline`函数做一些初始化跳板(trampoline)工作，这个映射函数map非常重要，我们会把它开盒，其将会依照我们前面所说的内存机制把物理页和虚拟页的映射做好。
```rust
  8     /// Mention that trampoline is not collected by areas.
  7     fn map_trampoline(&mut self) {
  6         self.page_table.map(
  5             VirtAddr::from(TRAMPOLINE).into(), // TRAMPOLINE的虚拟地址非常大，可以去查看。
  4             PhysAddr::from(strampoline as usize).into(), // stramploline位置在.text.entry之后，作为一个物理地址上的跳板
  3             PTEFlags::R | PTEFlags::X,
  2         );
  1     }
// -----------------------------------------------------------------
// for PageTable maps:
    6     /// set the map between virtual page number and physical page number
  5     #[allow(unused)]
  4     pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags) {
  3         let pte = self.find_pte_create(vpn).unwrap(); // 寻找或者给vpn创建相关的表项
  2         assert!(!pte.is_valid(), "vpn {:?} is mapped before mapping", vpn);
  1         *pte = PageTableEntry::new(ppn, flags | PTEFlags::V);
134     }
// -----------------------------------------------------------------
// for find_pte_create function.
 10     /// Find PageTableEntry by VirtPageNum, create a frame for a 4KB page table if not exist
 11     fn find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
 12         let idxs = vpn.indexes();
 13         let mut ppn = self.root_ppn; // root ppn.
 14         let mut result: Option<&mut PageTableEntry> = None;
            // Elegant method to take a trip in page table entries.
 15         for (i, idx) in idxs.iter().enumerate() {
                // idx: idx[0] -> idx[1] -> idx[2], high to low address.
 16             let pte = &mut ppn.get_pte_array()[*idx];
 17             if i == 2 {
 18                 result = Some(pte);
                    // 最后一个链路上应该存储的是真实的ppn，因此需要在这里break，在之后填入我们map函数中提供的ppn和flag信息。
 19                 break;
 20             }
 21             if !pte.is_valid() { // 看一看有没有valid项，没有的话就得新开一个。
 22                 let frame = frame_alloc().unwrap();
                    // 新创立一个PageTableEntry. ppn是frame分配时陪同存在的。
 23                 *pte = PageTableEntry::new(frame.ppn, PTEFlags::V);
 24                 // frames存放当前分配了空间的页表有哪些
                    self.frames.push(frame);
 25             }
 26             ppn = pte.ppn();
 27         }
 28         result
 29     }
 // -----------------------------------------------------------------
 // for vpn indexes, accumulate vpn using shift methods.
   6 impl VirtPageNum {
  5     /// Get the indexes of the page table entry
        /// idx[2] is the least significant part.
  4     pub fn indexes(&self) -> [usize; 3] {
  3         let mut vpn = self.0;
  2         let mut idx = [0usize; 3];
  1         for i in (0..3).rev() { // reverse the range, so we start countering from 2.
164             idx[i] = vpn & 511;
  1             vpn >>= 9;
  2         }
  3         idx
  4     }
  5 }
  // for physical page num.
    4 impl PhysPageNum {
  3     /// Get the reference of page table(array of ptes)
  2     pub fn get_pte_array(&self) -> &'static mut [PageTableEntry] {
  1         let pa: PhysAddr = (*self).into(); // line 152 in src/mm/address.rs.
            /// 将pa.0视作一个指向含有512个PTE项的数组的指针，之后fetch之即可，不管到底有没有这个数组存在，把他当做指针这一步均可以正常地进行。说白了512这个字你想想页表有多少个项就能马上明白是怎么一回事。
182         unsafe { core::slice::from_raw_parts_mut(pa.0 as *mut PageTableEntry, 512) }
  1     }
    }
  // for page table entry
    3 #[derive(Copy, Clone)]
  2 #[repr(C)]
  1 /// page table entry structure
25  pub struct PageTableEntry {
  1     /// bits of page table entry
  2     pub bits: usize,
  3 }


// -----------------------------------------------------------
// Good case for find_pte_create: no need to create a new entry.
// Bad case for find_pte_create
// 首先，按照64位经典模型，既然每个页大小是4KB，页表有512个项，也就说明了每个项的大小是8 Byte，特别的在这边`bits`恰好是usize，即8字节。
// RISC-V的页表项存两样东西，一样是ppn，一样是flags信息。
 12 impl PageTableEntry {
 11     /// Create a new page table entry
 10     pub fn new(ppn: PhysPageNum, flags: PTEFlags) -> Self {
  9         PageTableEntry {
  8             bits: ppn.0 << 10 | flags.bits as usize,
  7         }
  6     }
 25     /// Create an empty page table entry
 24     pub fn empty() -> Self {
 23         PageTableEntry { bits: 0 }
 22     }
 21     /// Get the physical page number from the page table entry
 20     pub fn ppn(&self) -> PhysPageNum {
 19         (self.bits >> 10 & ((1usize << 44) - 1)).into()
 18     }
 17     /// Get the flags from the page table entry
 16     pub fn flags(&self) -> PTEFlags {
 15         PTEFlags::from_bits(self.bits as u8).unwrap()
 14     }
 13     /// The page pointered by page table entry is valid?
 12     pub fn is_valid(&self) -> bool {
 11         (self.flags() & PTEFlags::V) != PTEFlags::empty()
 10     }
 }
 // for frame_alloc() function.
 impl FrameAllocator for StackFrameAllocator {
   1     fn alloc(&mut self) -> Option<PhysPageNum> {
            // 如果在recycle中，即刚刚dealloc释放，那么直接pop出来就好了
69          if let Some(ppn) = self.recycled.pop() {
  1             Some(ppn.into())
            // 没有空间了
  2         } else if self.current == self.end {
  3             None
            // 还有空间，且不是刚刚释放
  4         } else {
  5             self.current += 1;
  6             Some((self.current - 1).into())
  7         }
  8     }
 }
```
- 原linker加载后的文件的符号，在映射开启前的地址分布：
```rust
  4         // map kernel sections
  5         info!(".text [{:#x}, {:#x})", stext as usize, etext as usize);
  6         info!(".rodata [{:#x}, {:#x})", srodata as usize, erodata as usize);
  7         info!(".data [{:#x}, {:#x})", sdata as usize, edata as usize);
  8         info!(
  9             ".bss [{:#x}, {:#x})",
 10             sbss_with_stack as usize, ebss as usize
 11         );
 12         info!("mapping .text section");
 ```
- 在`memory_set`中压入各个段落的物理地址，在`push`的时候同时完成虚拟地址与物理地址的映射。
 ```rust
 13         memory_set.push(
 14             MapArea::new(
 15                 (stext as usize).into(),
 16                 (etext as usize).into(),
 17                 MapType::Identical,
 18                 MapPermission::R | MapPermission::X,
 19             ),
 20             None,
 21         );
 22         info!("mapping .rodata section");
 23         memory_set.push(
 24             MapArea::new(
 25                 (srodata as usize).into(),
 26                 (erodata as usize).into(),
 27                 MapType::Identical,
 28                 MapPermission::R,
 29             ),
 30             None,
 31         );
 32         info!("mapping .data section");
 33         memory_set.push(
 34             MapArea::new(
 35                 (sdata as usize).into(),
 36                 (edata as usize).into(),
  46                MapType::Identical,
 45                 MapPermission::R | MapPermission::W,
 44             ),
 43             None,
 42         );
 41         info!("mapping .bss section");
 40         memory_set.push(
 39             MapArea::new(
 38                 (sbss_with_stack as usize).into(),
 37                 (ebss as usize).into(),
 36                 MapType::Identical,
 35                 MapPermission::R | MapPermission::W,
 34             ),
 33             None,
 32         );
 31         info!("mapping physical memory");
 30         memory_set.push(
 29             MapArea::new(
 28                 (ekernel as usize).into(),
 27                 MEMORY_END.into(),
 26                 MapType::Identical,
 25                 MapPermission::R | MapPermission::W,
 24             ),
 23             None,
 22         );
 21         memory_set
 20     }
 ```
为了搞清楚这一点，我们需要对`memory_set`的`push`操作和`MapArea`的初始化操作做一些简单的分析，不过在此之前，我们简单看一下`MemorySet`的数据结构：

我们注意到，内存集由两个部分组成，其一是我们之前花了很大力气去描述的页表，其二即是这边需要记录的被映射的区域单元信息，`rCore`用`MapArea`来表示这一部分单元信息。
```rust
 36 /// address space
 35 pub struct MemorySet {
 34     page_table: PageTable,
 33     areas: Vec<MapArea>,
 32 }
    // in impl MemorySet:
  1     fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
  2         // MapArea.map function
            map_area.map(&mut self.page_table);
            // 反正目前似乎引来的都是None，无所谓。
  3         if let Some(data) = data {
  4             map_area.copy_data(&mut self.page_table, data);
  5         }
            // 评价为有个Vec在这里，所以该行为确实需要。
  6         self.areas.push(map_area);
  7     }
```
```rust
// MapArea.map
// 根据注释，我们知道MapArea用于控制虚拟内存的连续性片段
  1 /// map area structure, controls a contiguous piece of virtual memory
  2 pub struct MapArea {
  3     vpn_range: VPNRange,
  4     data_frames: BTreeMap<VirtPageNum, FrameTracker>,
  5     map_type: MapType,
  6     map_perm: MapPermission,
  7 }
  8
  9 impl MapArea {
        // 首先尝试解析上面提到的new的参数问题，还是比较直接的。
 10     pub fn new(
 11         start_va: VirtAddr,
 12         end_va: VirtAddr,
 13         map_type: MapType, // 共两种类型，Identical or Framed.
 14         map_perm: MapPermission, // RWXU存放权限的位置，对应作用不知和前述的页表有什么区别 -> 没有区别，不过记住一个流程，虚拟页建立时带着它的权限，再去寻找空闲的物理页，把二者的映射建立起来即可。
 15     ) -> Self {
 16         let start_vpn: VirtPageNum = start_va.floor();
 17         let end_vpn: VirtPageNum = end_va.ceil();
 18         Self {
 19             vpn_range: VPNRange::new(start_vpn, end_vpn), // 仅仅是用SimplerRange<VirtPageNum>这样的东东做了一个封装，表示虚拟内存所控制的范围。
 20             data_frames: BTreeMap::new(), // Rust特有的BTreeMap结构
 21             map_type, // 在作为参数时直接写入
 22             map_perm, // 在作为参数时直接写入
 23         }
 24     }
 35     pub fn map_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
            // 我们应该回去调用page_table.map函数（确实）
 34         let ppn: PhysPageNum;
            // 映射类型的检查，如果是Identical，似乎就不需要映射了，而是直接连接上，这是最朴素的方法了。
            // 如果是Framed类型，说明确实需要一个像样的映射方法，这时候就向FRAME_ALLOCATOR请求帮助，利用frame_alloc()搞来一个物理页就好了。
 33         match self.map_type {
 32             MapType::Identical => {
 31                 ppn = PhysPageNum(vpn.0);
 30             }
 29             MapType::Framed => {
 28                 let frame = frame_alloc().unwrap();
 27                 ppn = frame.ppn;
 26                 // 分配了Frames，要在MapArea中记录信息
                    self.data_frames.insert(vpn, frame);
 25             }
 24         }
            // 把flags搞到，准备进行映射
 23         let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
 22         page_table.map(vpn, ppn, pte_flags);
 21     }
 13     pub fn map(&mut self, page_table: &mut PageTable) {
            // 遍历一遍在vpn_range中出现过的全部虚拟页号。
 12         for vpn in self.vpn_range {
 11             self.map_one(page_table, vpn);
 10         }
  9     }
  }
```

#### 虚拟内存重映射测试：
对应的函数为`mm::remap_test()`。

------------------------------------------

------------------------------------------