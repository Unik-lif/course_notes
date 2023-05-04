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
#[no_mangle]
/// the rust entry-point of os
pub fn rust_main() -> ! {
    clear_bss();
    kernel_log_info();
    mm::init();
    println!("[kernel] back to world!");
    mm::remap_test();
    trap::init();
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    task::run_first_task();
    panic!("Unreachable in rust_main!");
}
```
#### 虚拟内存初始化：
虚拟内存的初始化主要分为两个部分，首先对堆进行初始化，其次对`frame_allocator`进行初始化，我们尝试逐一分析之。
```rust
/// initiate heap allocator, frame allocator and kernel space
pub fn init() {
    heap_allocator::init_heap();
    frame_allocator::init_frame_allocator();
    KERNEL_SPACE.exclusive_access().activate();
}
```
##### 堆初始化：
在堆初始化中，我们看到下面的代码结构。
```rust
#[global_allocator]
/// heap allocator instance
static HEAP_ALLOCATOR: LockedHeap = LockedHeap::empty();

#[alloc_error_handler]
/// panic when heap allocation error occurs
pub fn handle_alloc_error(layout: core::alloc::Layout) -> ! {
    panic!("Heap allocation error, layout = {:?}", layout);
}
/// heap space ([u8; KERNEL_HEAP_SIZE])
static mut HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];
/// initiate heap allocator
pub fn init_heap() {
    unsafe {
        HEAP_ALLOCATOR
            .lock()
            .init(HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE);
    }
}
```
与`Lab3`一样利用伙伴系统分配了一个大小为`KERNEL_HEAP_SIZE`的内存空间，在获取锁之后利用`init`对其进行操作，该操作其实是一个范式，能在相关的文档中找到。
##### 栈帧分配与初始化：
该函数的大体框架如图所示：
```rust
type FrameAllocatorImpl = StackFrameAllocator;

lazy_static! {
    /// frame allocator instance through lazy_static!
    pub static ref FRAME_ALLOCATOR: UPSafeCell<FrameAllocatorImpl> =
        unsafe { UPSafeCell::new(FrameAllocatorImpl::new()) };
}
/// initiate the frame allocator using `ekernel` and `MEMORY_END`
pub fn init_frame_allocator() {
    extern "C" {
        fn ekernel();
    }
    FRAME_ALLOCATOR.exclusive_access().init(
        PhysAddr::from(ekernel as usize).ceil(),
        PhysAddr::from(MEMORY_END).floor(),
    );
}
```
`ekernel`是内核镜像链接脚本中代表内核结束的位置，通过外部引用作为地址信息在函数`init_frame_allocator`中使用。

其中`FRAME_ALLOCATOR`是利用`UPSafeCell`包装好的内部可变的引用，这一点已经在前一个`Lab`中得到了阐述，而内部的单元则换了一层马甲，其实质如下所示。
```rust
 /// an implementation for frame allocator
 pub struct StackFrameAllocator {
     current: usize,
     end: usize,
     recycled: Vec<usize>,
 }
 impl StackFrameAllocator {
     pub fn init(&mut self, l: PhysPageNum, r: PhysPageNum) {
         self.current = l.0;
         self.end = r.0;
         // trace!("last {} Physical Frames.", self.end - self.current);
     }
 }
impl FrameAllocator for StackFrameAllocator {
     fn new() -> Self {
         Self {
             current: 0,
             end: 0,
             recycled: Vec::new(),
         }
     }
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
impl From<VirtAddr> for usize {
    fn from(v: VirtAddr) -> Self {
        if v.0 >= (1 << (VA_WIDTH_SV39 - 1)) {
            v.0 | (!((1 << VA_WIDTH_SV39) - 1))
        } else {
            v.0
        }
    }
}
```
这部分代码的核心理念，其实在于把`usize`转化成虚拟地址时针对第`38`位在`Sv39`中的特殊意义做了一些加工（`63 rd ~ 39 th bit` equals to `38 th bit`）。

下面这部分代码说白了就是根据`4 KB`的页大小考虑选取页有关`floor, ceil`的事宜，没什么好讲的。
```rust
/// virtual address impl
impl VirtAddr {
    /// Get the (floor) virtual page number
    pub fn floor(&self) -> VirtPageNum {
        VirtPageNum(self.0 / PAGE_SIZE)
    }

    /// Get the (ceil) virtual page number
    pub fn ceil(&self) -> VirtPageNum {
        VirtPageNum((self.0 - 1 + PAGE_SIZE) / PAGE_SIZE)
    }

    /// Get the page offset of virtual address
    pub fn page_offset(&self) -> usize {
        self.0 & (PAGE_SIZE - 1)
    }

    /// Check if the virtual address is aligned by page size
    pub fn aligned(&self) -> bool {
        self.page_offset() == 0
    }
}
```
----------------------------
我们继续分析原来的代码，因此可以推测下面这一串代码是以`ekernel`和`memory_end`地址作为物理页的最大范围，之后的虚拟内存和栈帧分配应该会是在这样一个物理页范围内执行相应的操作。
```rust
FRAME_ALLOCATOR.exclusive_access().init(
    PhysAddr::from(ekernel as usize).ceil(),
    PhysAddr::from(MEMORY_END).floor(),
);
```
##### KENREL_SPACE相应的操作
```rust
/// initiate heap allocator, frame allocator and kernel space
pub fn init() {
    heap_allocator::init_heap();
    frame_allocator::init_frame_allocator();
    KERNEL_SPACE.exclusive_access().activate();
} 
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
lazy_static! {
    /// The kernel's initial memory mapping(kernel address space)
    pub static ref KERNEL_SPACE: Arc<UPSafeCell<MemorySet>> =
        Arc::new(unsafe { UPSafeCell::new(MemorySet::new_kernel()) });
}
```
对于`MemorySet::new_kernel`这一函数，涉及的内容似乎还挺多的。我们尝试逐步分析。
- 初始化操作：
```rust
pub fn new_kernel() -> Self {
    // 利用new_bare建立一个空的、动态分配的MemorySet集合
    let mut memory_set = Self::new_bare();
    // map trampoline
    memory_set.map_trampoline();
```
在这一步中，利用函数`new_bare`建立一个空的集合。
```rust
/// Create a new empty `MemorySet`.
pub fn new_bare() -> Self {
    Self {
        page_table: PageTable::new(),
        areas: Vec::new(),
    }
}
```
之后，利用`map_trampoline`函数做一些初始化跳板(trampoline)工作，这个映射函数map非常重要，我们会把它开盒，其将会依照我们前面所说的内存机制把物理页和虚拟页的映射做好。
```rust
/// Mention that trampoline is not collected by areas.
fn map_trampoline(&mut self) {
    self.page_table.map(
        VirtAddr::from(TRAMPOLINE).into(), // TRAMPOLINE的虚拟地址非常大，可以去查看。
        PhysAddr::from(strampoline as usize).into(), // strampoline位置在.text.entry之后，作为一个物理地址上的跳板
        PTEFlags::R | PTEFlags::X,
    );
}
// -----------------------------------------------------------------
// for PageTable maps:
/// set the map between virtual page number and physical page number
#[allow(unused)]
pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags) {
    let pte = self.find_pte_create(vpn).unwrap(); // 寻找或者给vpn创建相关的表项
    assert!(!pte.is_valid(), "vpn {:?} is mapped before mapping", vpn);
    *pte = PageTableEntry::new(ppn, flags | PTEFlags::V);
}
// -----------------------------------------------------------------
// for find_pte_create function.
/// Find PageTableEntry by VirtPageNum, create a frame for a 4KB page table if not exist
fn find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
    let idxs = vpn.indexes();
    let mut ppn = self.root_ppn; // root ppn.
    let mut result: Option<&mut PageTableEntry> = None;
    // Elegant method to take a trip in page table entries.
    for (i, idx) in idxs.iter().enumerate() {
        // idx: idx[0] -> idx[1] -> idx[2], high to low address.
        let pte = &mut ppn.get_pte_array()[*idx];
        if i == 2 {
            result = Some(pte);
            // 最后一个链路上应该存储的是真实的ppn，因此需要在这里break，在之后填入我们map函数中提供的ppn和flag信息。
            break;
        }
        if !pte.is_valid() { // 看一看有没有valid项，没有的话就得新开一个。
            let frame = frame_alloc().unwrap();
            // 新创立一个PageTableEntry. ppn是frame分配时陪同存在的。
            *pte = PageTableEntry::new(frame.ppn, PTEFlags::V);
            // frames存放当前分配了空间的页表有哪些
            self.frames.push(frame);
        }
        ppn = pte.ppn();
    }
    result
}
 // -----------------------------------------------------------------
 // for vpn indexes, accumulate vpn using shift methods.
impl VirtPageNum {
    /// Get the indexes of the page table entry
    /// idx[2] is the least significant part.
    pub fn indexes(&self) -> [usize; 3] {
        let mut vpn = self.0;
        let mut idx = [0usize; 3];
        for i in (0..3).rev() { // reverse the range, so we start countering from 2.
            idx[i] = vpn & 511;
            vpn >>= 9;
        }
        idx
    }
}
  // for physical page num.
impl PhysPageNum {
    /// Get the reference of page table(array of ptes)
    pub fn get_pte_array(&self) -> &'static mut [PageTableEntry] {
        let pa: PhysAddr = (*self).into(); // line 152 in src/mm/address.rs.
        /// 将pa.0视作一个指向含有512个PTE项的数组的指针，之后fetch之即可，不管到底有没有这个数组存在，把他当做指针这一步均可以正常地进行。说白了512这个字你想想页表有多少个项就能马上明白是怎么一回事。
        unsafe { core::slice::from_raw_parts_mut(pa.0 as *mut PageTableEntry, 512) }
    }
}
// for page table entry
#[derive(Copy, Clone)]
#[repr(C)]
/// page table entry structure
pub struct PageTableEntry {
    /// bits of page table entry
    pub bits: usize,
}


// -----------------------------------------------------------
// Good case for find_pte_create: no need to create a new entry.
// Bad case for find_pte_create
// 首先，按照64位经典模型，既然每个页大小是4KB，页表有512个项，也就说明了每个项的大小是8 Byte，特别的在这边`bits`恰好是usize，即8字节。
// RISC-V的页表项存两样东西，一样是ppn，一样是flags信息。
impl PageTableEntry {
    /// Create a new page table entry
    pub fn new(ppn: PhysPageNum, flags: PTEFlags) -> Self {
        PageTableEntry {
            bits: ppn.0 << 10 | flags.bits as usize,
        }
    }
    /// Create an empty page table entry
    pub fn empty() -> Self {
        PageTableEntry { bits: 0 }
    }
    /// Get the physical page number from the page table entry
    pub fn ppn(&self) -> PhysPageNum {
        (self.bits >> 10 & ((1usize << 44) - 1)).into()
    }
    /// Get the flags from the page table entry
    pub fn flags(&self) -> PTEFlags {
        PTEFlags::from_bits(self.bits as u8).unwrap()
    }
    /// The page pointered by page table entry is valid?
    pub fn is_valid(&self) -> bool {
        (self.flags() & PTEFlags::V) != PTEFlags::empty()
    }
}
 // for frame_alloc() function.
impl FrameAllocator for StackFrameAllocator {
    fn alloc(&mut self) -> Option<PhysPageNum> {
       // 如果在recycle中，即刚刚dealloc释放，那么直接pop出来就好了
       if let Some(ppn) = self.recycled.pop() {
           Some(ppn.into())
       // 没有空间了
       } else if self.current == self.end {
           None
       // 还有空间，且不是刚刚释放
       } else {
           self.current += 1;
           Some((self.current - 1).into())
       }
   }
}
```
- 原linker加载后的文件的符号，在映射开启前的地址分布：
```rust
        // map kernel sections
        info!(".text [{:#x}, {:#x})", stext as usize, etext as usize);
        info!(".rodata [{:#x}, {:#x})", srodata as usize, erodata as usize);
        info!(".data [{:#x}, {:#x})", sdata as usize, edata as usize);
        info!(
            ".bss [{:#x}, {:#x})",
            sbss_with_stack as usize, ebss as usize
        );
        info!("mapping .text section");
 ```
- 在`memory_set`中压入各个段落的地址，在这边的`stext`和`etext`既是虚拟地址也是物理地址，不过在函数参数中其表意是虚拟地址，因为我们开启了`Identical`选项，在`push`的时候同时完成虚拟地址与物理地址的映射。
 ```rust
        memory_set.push(
            MapArea::new(
                (stext as usize).into(),
                (etext as usize).into(),
                MapType::Identical, // 直接映射之，维持一致
                MapPermission::R | MapPermission::X,
            ),
            None,
        );
        info!("mapping .rodata section");
        memory_set.push(
            MapArea::new(
                (srodata as usize).into(),
                (erodata as usize).into(),
                MapType::Identical,
                MapPermission::R,
            ),
            None,
        );
        info!("mapping .data section");
        memory_set.push(
            MapArea::new(
                (sdata as usize).into(),
                (edata as usize).into(),
                MapType::Identical,
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        info!("mapping .bss section");
        memory_set.push(
            MapArea::new(
                (sbss_with_stack as usize).into(),
                (ebss as usize).into(),
                MapType::Identical,
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        info!("mapping physical memory");
        memory_set.push(
            MapArea::new(
                (ekernel as usize).into(),
                MEMORY_END.into(),
                MapType::Identical, // 看上去分配出去了，实际上并没有实质内容
                MapPermission::R | MapPermission::W,
            ),
            None,
        );
        memory_set
    }
 ```
为了搞清楚这一点，我们需要对`memory_set`的`push`操作和`MapArea`的初始化操作做一些简单的分析，不过在此之前，我们简单看一下`MemorySet`的数据结构：

我们注意到，内存集由两个部分组成，其一是我们之前花了很大力气去描述的页表，其二即是这边需要记录的被映射的区域单元信息，`rCore`用`MapArea`来表示这一部分单元信息。
```rust
/// address space
pub struct MemorySet {
    page_table: PageTable,
    areas: Vec<MapArea>,
}
// in impl MemorySet:
fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
    // MapArea.map function
    map_area.map(&mut self.page_table);
    // 反正目前似乎引来的都是None，无所谓。
    if let Some(data) = data {
        map_area.copy_data(&mut self.page_table, data);
    }
    // 评价为有个Vec在这里，所以该行为确实需要。
    self.areas.push(map_area);
}
```
```rust
// MapArea.map
// 根据注释，我们知道MapArea用于控制虚拟内存的连续性片段
/// map area structure, controls a contiguous piece of virtual memory
pub struct MapArea {
    vpn_range: VPNRange,
    data_frames: BTreeMap<VirtPageNum, FrameTracker>,
    map_type: MapType,
    map_perm: MapPermission,
}

impl MapArea {
    // 首先尝试解析上面提到的new的参数问题，还是比较直接的。
    pub fn new(
        start_va: VirtAddr,
        end_va: VirtAddr,
        map_type: MapType, // 共两种类型，Identical or Framed.
        map_perm: MapPermission, // RWXU存放权限的位置，对应作用不知和前述的页表有什么区别 -> 没有区别，不过记住一个流程，虚拟页建立时带着它的权限，再去寻找空闲的物理页，把二者的映射建立起来即可。
    ) -> Self {
        let start_vpn: VirtPageNum = start_va.floor();
        let end_vpn: VirtPageNum = end_va.ceil();
        Self {
            vpn_range: VPNRange::new(start_vpn, end_vpn), // 仅仅是用SimplerRange<VirtPageNum>这样的东东做了一个封装，表示虚拟内存所控制的范围。
            data_frames: BTreeMap::new(), // Rust特有的BTreeMap结构
            map_type, // 在作为参数时直接写入
            map_perm, // 在作为参数时直接写入
        }
    }
    pub fn map_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
        // 我们应该回去调用page_table.map函数（确实）
        let ppn: PhysPageNum;
        // 映射类型的检查，如果是Identical，似乎就不需要映射了，而是直接连接上，这是最朴素的方法了。
        // 如果是Framed类型，说明确实需要一个像样的映射方法，这时候就向FRAME_ALLOCATOR请求帮助，利用frame_alloc()搞来一个物理页就好了。
        match self.map_type {
            MapType::Identical => {
                ppn = PhysPageNum(vpn.0);
            }
            MapType::Framed => {
                let frame = frame_alloc().unwrap();
                ppn = frame.ppn;
                // 分配了Frames，要在MapArea中记录信息
                self.data_frames.insert(vpn, frame);
            }
        }
        // 把flags搞到，准备进行映射
        let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
        page_table.map(vpn, ppn, pte_flags);
    }
    pub fn map(&mut self, page_table: &mut PageTable) {
        // 遍历一遍在vpn_range中出现过的全部虚拟页号。
        for vpn in self.vpn_range {
            self.map_one(page_table, vpn);
        }
    }
}
```
在完成了`KERNEL_SPACE`，即内核地址空间的初始化后，需要运行函数`activate`启动之。
```rust
pub fn activate(&self) {
    // 写入satp所需的token，即Sv39 | ppn信息
    // 实际上satp的后44位可以写任何PPN信息，这边用来存root_ppn
    let satp = self.page_table.token();
    unsafe {
        satp::write(satp);
        // 写完后需要运行sfence.vma指令以让全部处理器知道（说白了很像刷新一次），具体信息可查阅privileged手册。
        asm!("sfence.vma");
    }
}
```
#### 虚拟内存重映射测试：
对应的函数为`mm::remap_test()`。

说白了就是把先前塞进去的那一部分物理内存尝试重新拿过来，去找他们的虚拟地址的中间位置，然后去找对应的物理页是否有相应的被写入的特权级。
```rust
/// remap test in kernel space
#[allow(unused)]
pub fn remap_test() {
    let mut kernel_space = KERNEL_SPACE.exclusive_access();
    let mid_text: VirtAddr = ((stext as usize + etext as usize) / 2).into();
    let mid_rodata: VirtAddr = ((srodata as usize + erodata as usize) / 2).into();
    let mid_data: VirtAddr = ((sdata as usize + edata as usize) / 2).into();
    assert!(!kernel_space
        .page_table
        .translate(mid_text.floor())
        .unwrap()
        .writable(),);
    assert!(!kernel_space
        .page_table
        .translate(mid_rodata.floor())
        .unwrap()
        .writable(),);
    assert!(!kernel_space
        .page_table
        .translate(mid_data.floor())
        .unwrap()
        .executable(),);
    println!("remap_test passed!");
}
```

#### 中断Trap初始化
```rust
pub fn init() {
    set_kernel_trap_entry();
}

fn set_kernel_trap_entry() {
    unsafe {
        stvec::write(trap_from_kernel as usize, TrapMode::Direct);
    }
}
#[no_mangle]
/// handle trap from kernel
/// Unimplement: traps/interrupts/exceptions from kernel mode
/// Todo: Chapter 9: I/O device
pub fn trap_from_kernel() -> ! {
    use riscv::register::sepc;
    trace!("stval = {:#x}, sepc = {:#x}", stval::read(), sepc::read());
    panic!("a trap {:?} from kernel!", scause::read().cause());
}
```
这一步似乎只是把中断原因和返回地址之类的搞到，并没有做特别实质上的处理。
#### 计时器初始化
这一部分我们在`Lab3`中已经分析过了，跳过之。
```rust
/// enable timer interrupt in supervisor mode
pub fn enable_timer_interrupt() {
    unsafe {
        sie::set_stimer();
    }
}

/// Set the next timer interrupt
pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```
#### 任务运行
这是我们工作的另一个重点，对应的语句为
```rust
task::run_first_task();
```
同样使用内部可变的`TASK_MANGER`进行管理，
```rust
/// Run the first task in task list.
pub fn run_first_task() {
    TASK_MANAGER.run_first_task();
}
```
如下所示：
```rust
lazy_static! {
    /// a `TaskManager` global instance through lazy_static!
    pub static ref TASK_MANAGER: TaskManager = {
        println!("init TASK_MANAGER");
        // get_num_app利用先前提到的方法得到app的数目
        let num_app = get_num_app();
        println!("num_app = {}", num_app);
        // 声明一个tasks块，存放各个任务的控制块
        let mut tasks: Vec<TaskControlBlock> = Vec::new();
        for i in 0..num_app {
            // 初始化各个任务对应的TaskControlBlock
            tasks.push(TaskControlBlock::new(get_app_data(i), i));
        }
        // 返回之即可
        TaskManager {
            num_app,
            inner: unsafe {
                UPSafeCell::new(TaskManagerInner {
                    tasks,
                    current_task: 0,
                })
            },
        }
    };
}
```
`TaskControlBlock`块内新增加了很多特殊信息，简单查看一下。
```rust
/// The task control block (TCB) of a task.
pub struct TaskControlBlock {
    /// Save task context
    pub task_cx: TaskContext,

    /// Maintain the execution status of the current process
    pub task_status: TaskStatus,

    /// Application address space
    pub memory_set: MemorySet,

    /// The phys page number of trap context
    /// 内陷上下文存放的物理页号
    pub trap_cx_ppn: PhysPageNum,

    /// The size(top addr) of program which is loaded from elf file
    /// 程序基地址，这个应该是虚拟地址
    pub base_size: usize,

    /// Heap bottom
    /// 程序的堆地址底部位置（堆向上增长），同样也是虚拟地址
    pub heap_bottom: usize,

    /// Program break
    pub program_brk: usize,
}
```
此外，`TaskControlBlock`初始化的函数`new`也有一些改变，初始化似乎依赖我们的`ELF`信息，同时还要写入`app_id`信息

对于输入的`elf_data`数据，程序直接调用了函数`get_app_data(i)`来做，这个函数需要得到一些简单的分析：
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
上面这个函数似乎也没干什么，从`_num_app`处搞来的入口向量地址找到各个`APP`所在的地址，把他们对应的内容信息加载进来。

按照题目的要求，在`Lab4`中不再只是加载代码段信息的`binary form`，而是整个`ELF`文件进行装载。这是一件不错的事情。我们可以看到`src/link_app.S`文件内的部分信息，发现相关的应用程序以`elf`的形式读入了，所在的位置确乎是在数据段内`app_i_start`所示的位置。
```S
 .align 3
    .section .data
    .global _num_app
_num_app:
    .quad 11
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_3_start
    .quad app_4_start
    .quad app_5_start
    .quad app_6_start
    .quad app_7_start
    .quad app_8_start
    .quad app_9_start
    .quad app_10_start
    .quad app_10_end

    .section .data
    .global app_0_start
    .global app_0_end
    .align 3
app_0_start:
    .incbin "../user/build/elf/ch2b_bad_address.elf"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
    .align 3
app_1_start:
    .incbin "../user/build/elf/ch2b_bad_instructions.elf"
app_1_end:
```
对于`TaskControlBlock`块的初始化是复杂的，我们尝试分开来解读，如下所示：
```rust
pub fn new(elf_data: &[u8], app_id: usize) -> Self {
    // memory_set with elf program headers/trampoline/trap context/user stack
    // 通过ELF文件读取相关的地址空间、应用程序栈、ELF文件代码入口
    let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
    // TCB中一个载荷，trap处理跳板的物理页位置
    // 利用先前读取ELF文件所获得的memory_set，根据TRAP_CONTEXT_BASE寻找物理地址页
    let trap_cx_ppn = memory_set
        .translate(VirtAddr::from(TRAP_CONTEXT_BASE).into())
        .unwrap()
        .ppn();
    // 又一个载荷，任务当前运行的状态
    let task_status = TaskStatus::Ready;
    // map a kernel-stack in kernel space
    // 此处的bottom与top仅反映了地址上的底部和顶部，与真实环境下栈的样子不一样。
    let (kernel_stack_bottom, kernel_stack_top) = kernel_stack_position(app_id);
    KERNEL_SPACE.exclusive_access().insert_framed_area(
        kernel_stack_bottom.into(),
        kernel_stack_top.into(),
        MapPermission::R | MapPermission::W,
    );
    // 准备装载了
    let task_control_block = Self {
        task_status,
        task_cx: TaskContext::goto_trap_return(kernel_stack_top), // 应用程序的内核栈位置是kernel_stack_top，goto_trap_return返回task上下文
        memory_set,
        trap_cx_ppn,
        // 似乎这三个载荷在这边没有那么重要
        base_size: user_sp,
        heap_bottom: user_sp,
        program_brk: user_sp,
    };
    // prepare TrapContext in user space
    let trap_cx = task_control_block.get_trap_cx(); // trap_cx是mutable TrapContext
    *trap_cx = TrapContext::app_init_context(
        entry_point, // 进入地址
        user_sp, // 用户栈
        KERNEL_SPACE.exclusive_access().token(), // 这是pagetable存放的地方
        kernel_stack_top, // 内核栈
        trap_handler as usize, // trap_handler作为内陷处理函数的地址
    );
    task_control_block
}
```
##### ELF信息读取
我们首先来看从`ELF`文件中能够得到什么信息。回忆在做x86实验时的一些细节，`ELF`文件中比较重要的是利用`program header`去寻找重要的信息，参考下面的链接：

https://wiki.osdev.org/ELF

装载`ELF`的大体流程：
1. 检查`magic`数，验证是否合法
2. 阅读`ELF Header`，一个可运行的装载器往往仅关心`program headers`
3. 阅读`program headers`，他们可以帮助机器了解具体的程序段在文件的位置，以方便把代码段装载进来。
4. 解析`program headers`，确定哪些程序段是一定要装载进来的，只有`PT_LOAD`的段才是需要被装载进来的段。
5. 装载这些段，按照下面的步骤来做：首先，为每个代码段分配虚拟内存空间，从每个代码段的`p_vaddr`开始分配`p_memsz`大小的空间。其次，将段数据从`p_offset`位置开始的内容，复制到`p_vaddr`位置，这里复制的段大小是`p_filesz`。`p_memsz`和`p_filesz`的大小理应相同，如果不同则可以用`0`进行`padding`工作。
6. 读取可执行程序的进入地址。
7. 在装载好的内存空间中，跳转到进入地址并运行之。

从向量地址存储地读来应用程序的相关信息后，我们来看一下应该怎么处理它，这个函数名为`from_elf`。
```rust
/// Include sections in elf and trampoline and TrapContext and user stack,
/// also returns user_sp_base and entry point.
pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
    // 初始化空间
    let mut memory_set = Self::new_bare();
    // map trampoline，把跳板映射进来。strampoline是物理地址中对应的处理部分，对于用户的ELF，我们把虚拟地址中的trampoline映射到strampoline位置上。
    memory_set.map_trampoline();
    // map program headers of elf, with U flag
    let elf = xmas_elf::ElfFile::new(elf_data).unwrap();
    let elf_header = elf.header;
    let magic = elf_header.pt1.magic;
    // 检查可执行文件是否合法 - step 1.
    assert_eq!(magic, [0x7f, 0x45, 0x4c, 0x46], "invalid elf!");
    // 看看一共有多少个代码段 - step 2.
    let ph_count = elf_header.pt2.ph_count();
    // 这只是一个初始化，意义并不是特别大
    let mut max_end_vpn = VirtPageNum(0); 
    // step 3.
    for i in 0..ph_count {
        // program header information.
        let ph = elf.program_header(i).unwrap();
        // 找那些可以装载的代码段 - step 4.
        if ph.get_type().unwrap() == xmas_elf::program::Type::Load {
            // memsize always are larger than the filesize.
            let start_va: VirtAddr = (ph.virtual_addr() as usize).into();
            let end_va: VirtAddr = ((ph.virtual_addr() + ph.mem_size()) as usize).into();
            // U flag. for U-mode.
            let mut map_perm = MapPermission::U;
            let ph_flags = ph.flags();
            // Permission flag.
            if ph_flags.is_read() {
                map_perm |= MapPermission::R;
            }
            if ph_flags.is_write() {
                map_perm |= MapPermission::W;
            }
            if ph_flags.is_execute() {
                map_perm |= MapPermission::X;
            }
            // use framed, which means we will map it with multi-level page table.
            // Time to assemble!!
            // 内存映射，但是这一块区域其实是空的，什么东西也没有，需要把data写入
            let map_area = MapArea::new(start_va, end_va, MapType::Framed, map_perm);
            max_end_vpn = map_area.vpn_range.get_end();
            // copy_data -> see docs.
            // 在使用Framed方式读入时，需要同时装载数据以供应用程序使用（不如说data没写进去，这玩意是空白的，没有意义）
            // 或许得要有更多的x86编程经验才比较方便理解这一点（×），看一看ELF就好了哈哈
            // 代码需要被读入
            memory_set.push(
                map_area,
                Some(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize]),
            );
        }
    }
    // map user stack with U flags
    // 这里其实比较奇怪。
    // https://docs.oracle.com/cd/E19683-01/816-1386/chapter6-83432/index.html
    // 找到解答了捏
    let max_end_va: VirtAddr = max_end_vpn.into();
    let mut user_stack_bottom: usize = max_end_va.into();
    // 现在在最高地址位置层了
    // guard page
    user_stack_bottom += PAGE_SIZE;
    // guard page之上是用户的栈空间，大小为两个4 KB页面
    let user_stack_top = user_stack_bottom + USER_STACK_SIZE;
    // 将用户栈利用Framed方式写入内存集合中
    memory_set.push(
        MapArea::new(
            user_stack_bottom.into(),
            user_stack_top.into(),
            MapType::Framed,
            MapPermission::R | MapPermission::W | MapPermission::U,
        ),
        None,
    );
    // used in sbrk，这个真的不清楚干什么用的。姑且就认为在用户栈之上还有个提供给sbrk系统调用的空间吧。
    memory_set.push(
        MapArea::new(
            user_stack_top.into(),
            user_stack_top.into(),
            MapType::Framed,
            MapPermission::R | MapPermission::W | MapPermission::U,
        ),
        None,
    );
    // map TrapContext，在次高位设置了一个用于存放Trap上下文信息的内存空间，并以Framed的方式进行管理。
    memory_set.push(
        MapArea::new(
            TRAP_CONTEXT_BASE.into(),
            TRAMPOLINE.into(),
            MapType::Framed,
            MapPermission::R | MapPermission::W,
        ),
        None,
    );
    (
        memory_set,
        user_stack_top,
        elf.header.pt2.entry_point() as usize,
    )
}

```


该函数的最终目的是实现如下的应用程序内存排布：

![](http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/app-as-full.png)

经过这一部分的解析，我们初始化了某个应用程序的内存集合`Memory_set`排布方式（如上面的图片所述），确定了其用于栈和处理`trap`的跳板位置，同时还返回了`ELF`文件中代码文件的入口位置。

##### 内核栈空间寻找
在内核中的内存空间排布如下图所示：前者是内核高地址的排布，后者是内核低地址的排布。

![](http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/kernel-as-high.png)

![](http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/kernel-as-low.png)

我们分析函数：`kernel_stack_position`的相关情况
```rust
/// Return (bottom, top) of a kernel stack in kernel space.
pub fn kernel_stack_position(app_id: usize) -> (usize, usize) {
    let top = TRAMPOLINE - app_id * (KERNEL_STACK_SIZE + PAGE_SIZE);
    let bottom = top - KERNEL_STACK_SIZE;
    (bottom, top)
}
```
这个函数根据应用程序的`ID`返回了应用程序在内核空间上的内核栈位置，左底右顶（但并非栈底和栈顶），显然是从高地址`TRAMPOLINE`位置往下减，而`KERNEL_STACK_SIZE`也很恰好是两个页的大小。
##### TrapContext内容
`TrapContext`拥有丰富的含义，如下所示：
```rust
#[repr(C)]
#[derive(Debug)]
/// trap context structure containing sstatus, sepc and registers
pub struct TrapContext {
    /// General-Purpose Register x0-31
    pub x: [usize; 32],
    /// Supervisor Status Register
    pub sstatus: Sstatus,
    /// Supervisor Exception Program Counter
    pub sepc: usize,
    /// Token of kernel address space
    pub kernel_satp: usize,
    /// Kernel stack pointer of the current application
    pub kernel_sp: usize,
    /// Virtual address of trap handler entry point in kernel
    /// cite a function in fact.
    pub trap_handler: usize,
}
```
其内部的两个函数如下所示，信息在注释中已经解释得很丰富，我们不再参考之：
```rust
impl TrapContext {
    /// put the sp(stack pointer) into x\[2\] field of TrapContext
    pub fn set_sp(&mut self, sp: usize) {
        self.x[2] = sp;
    }
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
}
```

到这里，我们完成了`TCB`块初始化的解读，`TASK_MANAGER`的初始化也就自然迎刃而解了，接下来是要让程序能够跑起来了。
##### 开始运行第一个任务
经典采用`_unused`任务进行替罪羊的一个当。
```rust
fn run_first_task(&self) -> ! {
    let mut inner = self.inner.exclusive_access();
    // 攫取第一个任务
    let next_task = &mut inner.tasks[0];
    // 设置任务状态为Running
    next_task.task_status = TaskStatus::Running;
    let next_task_cx_ptr = &next_task.task_cx as *const TaskContext; // 注意先前初始化写入的一些数据
    drop(inner);
    let mut _unused = TaskContext::zero_init();
    // before this, we should drop local variables that must be dropped manually
    unsafe {
        __switch(&mut _unused as *mut _, next_task_cx_ptr);
    }
    panic!("unreachable in run_first_task!");
}
```
利用`__switch`在多个任务之间进行切换。
```rust
core::arch::global_asm!(include_str!("switch.S"));
use super::TaskContext;

extern "C" {
    /// Switch to the context of `next_task_cx_ptr`, saving the current context
    /// in `current_task_cx_ptr`.
    pub fn __switch(current_task_cx_ptr: *mut TaskContext, next_task_cx_ptr: *const TaskContext);
}
```
`__switch`函数如下所示，其拥有两个参数，其一为当前运行的任务的上下文指针，其二为接下来运行的任务的上下文指针。

其实说起来也确实很简单，`a0`和`a1`分别反映二者任务存放上下文的指针地址，只要进行`load store`操作就完事了。
```rust
__switch:
    # __switch(
    #     current_task_cx_ptr: *mut TaskContext,
    #     next_task_cx_ptr: *const TaskContext
    # )
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    ret
```
最后的`ret`将会返回到`ra`所存储的位置，在先前的初始化中，我们上下文中写入的第一个参数，即`ra`恰好是`trap_return`函数（可以查看`goto_trap_return`函数）。
```rust
#[no_mangle]
/// return to user space
/// set the new addr of __restore asm function in TRAMPOLINE page,
/// set the reg a0 = trap_cx_ptr, reg a1 = phy addr of usr page table,
/// finally, jump to new addr of __restore asm function
pub fn trap_return() -> ! {
    // 确定入口
    set_user_trap_entry();
    let trap_cx_ptr = TRAP_CONTEXT_BASE;
    // user page table physical address.
    let user_satp = current_user_token();
    extern "C" {
        fn __alltraps();
        fn __restore();
    }
    // restore_va函数的虚拟地址所在位置
    let restore_va = __restore as usize - __alltraps as usize + TRAMPOLINE;
    // trace!("[kernel] trap_return: ..before return");
    unsafe {
        asm!(
            "fence.i",
            "jr {restore_va}",         // jump to new addr of __restore asm function
            restore_va = in(reg) restore_va,
            in("a0") trap_cx_ptr,      // a0 = virt addr of Trap Context
            in("a1") user_satp,        // a1 = phy addr of usr page table
            options(noreturn)
        );
    }
}
```
函数`set_user_trap_entry`如下所示：
```rust
fn set_user_trap_entry() {
    unsafe {
        stvec::write(TRAMPOLINE as usize, TrapMode::Direct); // stvec: trap处理代码的位置在TRAMPOLINE
    }
}
```
上面的函数`trap_return`调用了`__restore`函数，并且用内陷上下文虚拟地址和用户空间的页表基地址作为参数输入。
```rust
__restore:
    # a0: *TrapContext in user space(Constant); a1: user space token
    # switch to user space
    csrw satp, a1
    sfence.vma # clear the tlb
    csrw sscratch, a0
    mv sp, a0
    # now sp points to TrapContext in user space, start restoring based on it
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    csrw sstatus, t0 # sstatus
    csrw sepc, t1 # entry_point of app
    # restore general purpose registers except x0/sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # back to user stack
    ld sp, 2*8(sp)
    sret
```
这个函数主要做了下面这些事情：
1. 修改`satp`的`token`以实现从内核空间到用户空间的转换，清空`TLB`。
2. 让`sp`指向用户空间中的`TrapContext`，并根据`TrapContext`装载上下文。
3. 切换到先前写好的用户栈上，这个栈其实就是`user_sp`，这个信息是在`from_elf`的函数在用户空间开出来的一块区域。
4. 返回到用户态上，入口地址为`sepc`，恰好就是`ELF`程序的入口。

> 而当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 S 特权级的特权指令 sret 来完成，这一条指令具体完成以下功能：
> 
> CPU 会将当前的特权级按照 sstatus 的 SPP 字段设置为 U 或者 S ；
> 
> CPU 会跳转到 sepc 寄存器指向的那条指令，然后继续执行。

运行完第一个`ELF`程序后，照例会出现系统调用等等，这时候应该去看看内陷处理器怎么做了。

还记的我们之前写的`TRAMPOLINE`吗？这玩意是怎么工作的？为什么我们能够做到指哪打哪？

首先，这一部分代码是如何装载进来的？

在`linker.ld`中看到`trampoline`代码被装载到了`strampoline`的物理内存位置。对应的我们又能找到这个段所在的位置。
```rust
strampoline = .;
*(.text.trampoline);
. = ALIGN(4K);
*(.text .text.*)

//trap.S
.section .text.trampoline
```
之后，将`TRAMPOLINE`和物理地址`strampoline`映射在一起。
```rust
/// Mention that trampoline is not collected by areas.
fn map_trampoline(&mut self) {
    self.page_table.map(
        VirtAddr::from(TRAMPOLINE).into(),
        PhysAddr::from(strampoline as usize).into(),
        PTEFlags::R | PTEFlags::X,
    );
}
```
所以之后只要把`TRAMPOLINE`写进`stvec`中就可以了。
```rust
  2 fn set_user_trap_entry() {
  1     unsafe {
45          stvec::write(TRAMPOLINE as usize, TrapMode::Direct);
  1     }
  2 }
```
进入后，将会运行`__alltraps`汇编代码，然后再进入`trap_handler`之中。
```S
__alltraps:
    csrrw sp, sscratch, sp
    # now sp->*TrapContext in user space, sscratch->user stack
    # save other general purpose registers
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
    # we can use t0/t1/t2 freely, because they have been saved in TrapContext
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it in TrapContext
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # load kernel_satp into t0
    ld t0, 34*8(sp)
    # load trap_handler into t1
    ld t1, 36*8(sp)
    # move to kernel_sp
    ld sp, 35*8(sp)
    # switch to kernel space
    csrw satp, t0
    sfence.vma
    # jump to trap_handler
    jr t1
```
这个名为`__alltraps`的函数主要做了下面的这些事情：
1. 将内陷上下文存放到`sp`之中，而原本的`sp`，即用户栈将会存放到`sscratch`寄存器中
2. 将当前的上下文信息存放到内陷上下文的位置，并对相关的特权级寄存器信息进行读取，以便进入内核态
3. 清空`TLB`缓存，进入`trap_handler`
```rust
/// trap handler
#[no_mangle]
pub fn trap_handler() -> ! {
    // 防止内核出现陷入，给它送一个处理函数
    set_kernel_trap_entry();
    // 利用TASK_MANAGER获取当前的内陷上下文对应的PPN
    let cx = current_trap_cx();
    let scause = scause::read(); // get trap cause
    let stval = stval::read(); // get extra value
    // trace!("into {:?}", scause.cause());
    match scause.cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            // jump to next instruction anyway
            cx.sepc += 4;
            // get system call return value
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
        }
        Trap::Exception(Exception::StoreFault)
        | Trap::Exception(Exception::StorePageFault)
        | Trap::Exception(Exception::LoadFault)
        | Trap::Exception(Exception::LoadPageFault) => {
            println!("[kernel] PageFault in application, bad addr = {:#x}, bad instruction = {:#x}, kernel killed it.", stval, cx.sepc);
            exit_current_and_run_next();
        }
        Trap::Exception(Exception::IllegalInstruction) => {
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            exit_current_and_run_next();
        }
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            set_next_trigger();
            suspend_current_and_run_next();
        }
        _ => {
            panic!(
                "Unsupported trap {:?}, stval = {:#x}!",
                scause.cause(),
                stval
            );
        }
    }
    //println!("before trap_return");
    trap_return();
}
```
简单阐述步骤如下：
1. 利用函数`set_kernel_trap_entry`防止内核态中断的情况。在处理`trap`的时候，我们此时的CPU处于内核态，要警惕内核出现异常，如果有，采用`trap_from_kernel`函数来`handle`它。
```rust
fn set_kernel_trap_entry() {
    unsafe {
        stvec::write(trap_from_kernel as usize, TrapMode::Direct);
    }
}
```
2. 从寄存器中获取内陷的原因，利用`TASK_MANAGER`找到各个应用程序的`Trap_cx`，这玩意就虚拟地址上是存放在次高页的，先前已经从`Memory_set`搞来了它的`PPN`了
3. 根据不同的原因分发处理即可

根据不同系统调用进行处理的流程参考`lab3.md`，考虑到篇幅，我们的代码跟踪活动就不再赘述了（其实真的没有太大必要，不是非常相像，是完全一致，不同的地方仅仅是`trap`过程和恢复等等），接下来请参考文档。

------------------------------------------
### 文档阅读补充：
1. FrameTracker：RAII机制的体现，用于给物理地址做封装，以实现生命周期的管理。封装后可以利用trait特性让物理地址页自动被drop以实现回收。这样就不需要程序员手动将堆上分配的资源回收了。
2. 考虑到页表的数据结构处理方式：
```rust
pub struct PageTable {
    root_ppn: PhysPageNum,
    frames: Vec<FrameTracker>,
}

impl PageTable {
    pub fn new() -> Self {
        let frame = frame_alloc().unwrap();
        PageTable {
            root_ppn: frame.ppn,
            frames: vec![frame],
        }
    }
}
```
FrameTracker的生命周期进一步被绑定在PageTable之上，在PageTable被释放的时候，FrameTracker及其内部存放的物理地址页同样会被自动释放掉。
3. 逻辑段MapArea用于表示连续地址的虚拟内存，所谓逻辑段，就是指地址区间中的一段实际可用（即 MMU 通过查多级页表可以正确完成地址转换）的地址连续的虚拟地址区间，该区间内包含的所有虚拟页面都以一种相同的方式映射到物理页帧，具有可读/可写/可执行等属性。

### 流程梳理：
1. 创建内核地址空间，即我们上面的`mm::init()`，运行`page_table.token()`，将特殊信息写给`satp`，以开启分页（开启前需要清空TLB内的键值对，使用`sfence.vma`作为内存屏障）。

特别的，这里提到了一个细节值得注意：
> 幸运的是，我们做到了这一点。这条写入 satp 的指令及其下一条指令都在内核内存布局的代码段中，在切换之后是一个恒等映射，而在切换之前是视为物理地址直接取指，也可以将其看成一个恒等映射。这完全符合我们的期待：即使切换了地址空间，指令仍应该能够被连续的执行。

2. 利用`remap_test`来检查内核地址空间的多级页表是否被正确地设置了。
3. 在分页机制被使能之后，实现跳板机制变得复杂起来，需要让应用和内核地址空间在切换地址空间指令附近是平滑的，需要在切换的过程中同时完成地址空间的切换，即对`satp`寄存器完成修改。`rCore`采用了`xv6`里的隔离方法，让内核与应用程序的地址空间是相互隔离的。

切换过程需要两个信息，一是内核地址空间的`token`，二是应用的内核栈栈顶的位置。我们仅有一个sscratch寄存器可以拿来用，因此不得不将`Trap`上下文保存在应用地址空间的一个虚拟页面中，而不是切换到内核地址空间去保存在内核栈上。

说白了这个`token`能反映`root_ppn`的位置呗，没什么好评价的，在切换进程时估计用到挺多的。
```rust
/// get the token from the page table
pub fn token(&self) -> usize {
    8usize << 60 | self.root_ppn.0 // 这边开启的确乎是Sv39.
}
```
4. 跳板机制的源码：请参考文档，写的不错，我感觉我没什么好评价的，细节多到爆炸，我就不应该直接碰源码。

> 问题的本质可以概括为：跳转指令实际被执行时的虚拟地址和在编译器/汇编器/链接器进行后端代码生成和链接形成最终机器码时设置此指令的地址是不同的。

这句话很酷哦。

### 本次源码阅读到此结束
**难点：**
1. 内核的虚拟地址排布与相关复杂的数据结构
2. `elf`的解读