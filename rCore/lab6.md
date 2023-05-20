## 第六章：文件系统
`easyfs`文件系统的框架：
1. 磁盘块设备接口层：读写磁盘块设备的`trait`接口
2. 块缓存层：位于内存的磁盘块数据缓存
3. 磁盘数据结构层：表示磁盘文件系统的数据结构
4. 磁盘块管理层：实现对磁盘文件系统的管理
5. 索引节点层：实现文件创建，文件打开，文件读写等操作

### easyfs系统
这个系统似乎是以库函数的方式管理的。

## 读源码：OS 部分
相比于前一次的`Lab`，本次的`Lab`添加了文件系统这一玩意儿，我们简单追溯一下。
```rust
fs::list_apps();


lazy_static! {
    pub static ref ROOT_INODE: Arc<Inode> = {
        let efs = EasyFileSystem::open(BLOCK_DEVICE.clone());
        Arc::new(EasyFileSystem::root_inode(&efs))
    };
}

/// List all apps in the root directory
pub fn list_apps() {
    println!("/**** APPS ****");
    for app in ROOT_INODE.ls() {
        println!("{}", app);
    }
    println!("**************/");
}
```
需要对`ROOT_INODE`做一下研究，其中的`Inode`类型我们尚是不清楚的。
```rust
/// Virtual filesystem layer over easy-fs
pub struct Inode {
    block_id: usize,
    block_offset: usize,
    fs: Arc<Mutex<EasyFileSystem>>,
    block_device: Arc<dyn BlockDevice>,
}
```
根据文档信息，文件系统的使用者往往不关心磁盘布局是如何实现的，而是更希望能够直接看到目录树结构中逻辑上的文件和目录。`Inode`本质上和磁盘并没有太大联系，它只是一个位于内存中的管理用的数据结构，其中包含了`Inode`对应的`DiskInode`保存在磁盘中的具体位置，如`block_id`和`block_offset`这一部分，`fs`则指向文件系统的位置，因为对于`Inode`的种种操作需要通过文件系统来进行。最后一个载荷`block_device`对应着相关的块设备位置。

`EasyFileSystem`中的`open`与`BLOCK_DEVICE`是何用意？
```rust
    pub fn open(block_device: Arc<dyn BlockDevice>) -> Arc<Mutex<Self>> {
        // read SuperBlock
        // 我们检索是否有块号为0的block cache存在，不存在就新建一个了事。
        // 这边的核心工作是找到块号为0的block cache，而它往往其实就是SuperBlock，是专门用来管理其他块的东东
        get_block_cache(0, Arc::clone(&block_device))
            .lock()
            // 特定位置的块内存地址，被以&SuperBlock的类型进行解析了。
            .read(0, |super_block: &SuperBlock| {
                // valid number.
                assert!(super_block.is_valid(), "Error loading EFS!");
                // 共有多少个inode块
                let inode_total_blocks =
                    super_block.inode_bitmap_blocks + super_block.inode_area_blocks;
                let efs = Self {
                    block_device,
                    inode_bitmap: Bitmap::new(1, super_block.inode_bitmap_blocks as usize),
                    data_bitmap: Bitmap::new(
                        (1 + inode_total_blocks) as usize,
                        super_block.data_bitmap_blocks as usize,
                    ),
                    // 这边仅是一个简单的计算
                    inode_area_start_block: 1 + super_block.inode_bitmap_blocks,
                    data_area_start_block: 1 + inode_total_blocks + super_block.data_bitmap_blocks,
                };
                Arc::new(Mutex::new(efs))
            })
    }
```
`open`函数从给定的块设备处，读取得到了块`cache`信息。函数`get_block_cache`是从块`cache`管理者处根据`block_id`和`block_device`读取相关的信息块。
```rust
lazy_static! {
    /// The global block cache manager
    pub static ref BLOCK_CACHE_MANAGER: Mutex<BlockCacheManager> =
        Mutex::new(BlockCacheManager::new());
}
/// Get the block cache corresponding to the given block id and block device
pub fn get_block_cache(
    block_id: usize,
    block_device: Arc<dyn BlockDevice>,
) -> Arc<Mutex<BlockCache>> {
    BLOCK_CACHE_MANAGER
        .lock()
        .get_block_cache(block_id, block_device)
}
```
`BLOCK_CACHE_MANAGER`是以一种最微不足道的方式进行初始化的：`new`函数和`get_block_cache`函数如下所示。
```rust
    pub fn new() -> Self {
        Self {
            queue: VecDeque::new(),
        }
    }

    pub fn get_block_cache(
        &mut self,
        block_id: usize,
        block_device: Arc<dyn BlockDevice>,
    ) -> Arc<Mutex<BlockCache>> {
        // 遍历当前的BLOCK_CACHE_MANAGER寻找对应的块设备，如果block_id是对的，那皆大欢喜，并返回。
        if let Some(pair) = self.queue.iter().find(|pair| pair.0 == block_id) {
            Arc::clone(&pair.1)
        }
        // 倘若不巧，没有找到，则执行下列操作 
        else {
            // substitute，没有找到，那没办法我们需要把Vecqueue中没有被多次引用的一个东东找出来丢掉，以方便我们后续新添加block cache。
            if self.queue.len() == BLOCK_CACHE_SIZE {
                // from front to tail
                if let Some((idx, _)) = self
                    .queue
                    .iter()
                    .enumerate()
                    .find(|(_, pair)| Arc::strong_count(&pair.1) == 1)
                {
                    self.queue.drain(idx..=idx);
                } else {
                    panic!("Run out of BlockCache!");
                }
            }
            // 就假设没这号人了吧，不存在的话，我们新开一个块Cache就好了，将这个块设备和block_id绑定在一起
            // load block into mem and push back
            let block_cache = Arc::new(Mutex::new(BlockCache::new(
                block_id,
                Arc::clone(&block_device),
            )));
            self.queue.push_back((block_id, Arc::clone(&block_cache)));
            block_cache
        }
    }
```
块中的`Cache`如图所示，其初始化流程一并在此给出：当然，这个流程似乎真没什么好说的。
```rust
/// Cached block inside memory
pub struct BlockCache {
    /// cached block data
    cache: [u8; BLOCK_SZ],
    /// underlying block id
    block_id: usize,
    /// underlying block device
    block_device: Arc<dyn BlockDevice>,
    /// whether the block is dirty
    modified: bool,
}

    pub fn new(block_id: usize, block_device: Arc<dyn BlockDevice>) -> Self {
        let mut cache = [0u8; BLOCK_SZ];
        block_device.read_block(block_id, &mut cache);
        Self {
            cache,
            block_id,
            block_device,
            modified: false,
        }
    }
```
我们之后再来解析一下函数`read`，这个函数相对来说还是比较复杂的。该函数是`BlockCache`特有的可以使用的函数，`read`函数的实质是读取`Cache`对应的内存位置，并且针对相关的函数闭包解析相关的地址信息，并以此进行一些函数操作。
```rust
    /// Get the address of an offset inside the cached block data
    fn addr_of_offset(&self, offset: usize) -> usize {
        &self.cache[offset] as *const _ as usize
    }

    pub fn get_ref<T>(&self, offset: usize) -> &T
    where
        T: Sized,
    {
        let type_size = core::mem::size_of::<T>();
        // 这里的T似乎不好梳理，得考虑清楚。
        // 参考文档：
        /* get_ref 是一个泛型方法，它可以获取缓冲区中的位于偏移量 offset 的一个类型为 T 的磁盘上数据结构的不可变引用。
         * 该泛型方法的 Trait Bound 限制类型 T 必须是一个编译时已知大小的类型，
         * 我们通过 core::mem::size_of::<T>() 在编译时获取类型 T 的大小并确认该数据结构被整个包含在磁盘块及其缓冲区之内。
         * 这里编译器会自动进行生命周期标注，约束返回的引用的生命周期不超过 BlockCache 自身，在使用的时候我们会保证这一点。
        */
        assert!(offset + type_size <= BLOCK_SZ);
        let addr = self.addr_of_offset(offset);
        unsafe { &*(addr as *const T) }
    }

    pub fn get_mut<T>(&mut self, offset: usize) -> &mut T
    where
        T: Sized,
    {
        let type_size = core::mem::size_of::<T>();
        assert!(offset + type_size <= BLOCK_SZ);
        self.modified = true;
        let addr = self.addr_of_offset(offset);
        unsafe { &mut *(addr as *mut T) }
    }

    pub fn read<T, V>(&self, offset: usize, f: impl FnOnce(&T) -> V) -> V {
        f(self.get_ref(offset))
    }

    pub fn modify<T, V>(&mut self, offset: usize, f: impl FnOnce(&mut T) -> V) -> V {
        f(self.get_mut(offset))
    }
```
函数`addr_of_offset`显然是`cache`地址获取的操作，根据`offset`所对应的类型计算具体的内存地址位置。现在，由于闭包内参数的类型是`&SuperBlock`，这意味着我们的地址以这种方式被解析了！我们需要了解`SuperBlock`的相关数据结构。不过在读源码的时候我有一个悬而未决的问题，最早的`SuperBlock`到底是什么时候被创造`Create`出来的？在`OS`相关的源码中我并没有找到。
```rust
/// Super block of a filesystem
#[repr(C)]
pub struct SuperBlock {
    magic: u32, // valid or not.
    pub total_blocks: u32, // 
    pub inode_bitmap_blocks: u32,
    pub inode_area_blocks: u32,
    pub data_bitmap_blocks: u32,
    pub data_area_blocks: u32,
}
```
根据文档，`easy-fs`被写入一个块设备中，`OS`无需对其进行`Create`初始化操作，仅仅需要做到`Open`打开之。因此上面的`open`函数若运行正常将很轻松地打开`superblock`，这就解答了我们的疑问，即`magic number`是什么时候写入的。

后续对于`inode_bitmap`和`data_bitmap`的初始化，我们简单分析如下：
```rust
    /// A new bitmap from start block id and number of blocks
    pub fn new(start_block_id: usize, blocks: usize) -> Self {
        Self {
            start_block_id,
            blocks,
        }
    }
```
根据块号和对应的块们进行初始化，似乎也没什么好说的。很自然的。

之后，我们来看一下`root_node`的初始化：利用了我们`Open`时建立的`efs`文件
```rust
Arc::new(EasyFileSystem::root_inode(&efs))

    /// Get the root inode of the filesystem
    pub fn root_inode(efs: &Arc<Mutex<Self>>) -> Inode {
        // 获取块设备
        let block_device = Arc::clone(&efs.lock().block_device);
        // acquire efs lock temporarily，找到0号对应的位置
        let (block_id, block_offset) = efs.lock().get_disk_inode_pos(0);
        // release efs lock
        // 这样就能得到最终我们想要的Inode了
        Inode::new(block_id, block_offset, Arc::clone(efs), block_device)
    }

    /// Get inode by id
    /// bitmap存放的是映射，area部分存放的才是真实的inode_disk相关信息
    pub fn get_disk_inode_pos(&self, inode_id: u32) -> (u32, usize) {
        // 真实场景下DiskInode的大小
        let inode_size = core::mem::size_of::<DiskInode>();
        // 每个block有多少个Inode
        let inodes_per_block = (BLOCK_SZ / inode_size) as u32;
        // 希望确认block号，从start_block开始
        let block_id = self.inode_area_start_block + inode_id / inodes_per_block;
        (
            block_id,
            // 第二个返回值应该是块内的偏移量
            (inode_id % inodes_per_block) as usize * inode_size,
        )
    }
```
至此，`OS`部分的文件系统就似乎分析完成了？我们之后或许看文档按需扩充似乎更加合适？
## 块设备结构
### 最底层：块设备接口BlockDevice
```rust
use core::any::Any;
/// Trait for block devices
/// which reads and writes data in the unit of blocks
pub trait BlockDevice: Send + Sync + Any {
    ///Read data form block to buffer
    fn read_block(&self, block_id: usize, buf: &mut [u8]);
    ///Write data from buffer to block
    fn write_block(&self, block_id: usize, buf: &[u8]);
}
```
在最底层定义了`BlockDevice`这一特征，要求满足`read_block`和`write_block`函数的具体实现。
### 第二层：块缓存Block Cache
我们分割出一部分内存，用于当做磁盘的缓存。缓存信息被写在`cache`这一载荷处，其最大能够把整个`Block`中的信息进行缓存。块设备如您所见，是一个满足了特征`BlockDevice`的东东。
```rust
/// Cached block inside memory
pub struct BlockCache {
    /// cached block data
    cache: [u8; BLOCK_SZ],
    /// underlying block id
    block_id: usize,
    /// underlying block device
    block_device: Arc<dyn BlockDevice>,
    /// whether the block is dirty
    modified: bool,
}
```
`BlockCache`所提供的一些基础方法我们已经在上面提及过了，特别的，块`Cache`所对应的块类型和这一类型的数据结构也要同时存放在缓存和块本身中，因此不能判别时不能超过块的大小。
```rust
    pub fn get_ref<T>(&self, offset: usize) -> &T
    where
        T: Sized,
    {
        let type_size = core::mem::size_of::<T>();
        // type本身也要在块中存储，要满足这个判别式，即我的offset不可以太大，要给块类型的存储留下空间。
        assert!(offset + type_size <= BLOCK_SZ);
        let addr = self.addr_of_offset(offset);
        unsafe { &*(addr as *const T) }
    }

    pub fn get_mut<T>(&mut self, offset: usize) -> &mut T
    where
        T: Sized,
    {
        let type_size = core::mem::size_of::<T>();
        assert!(offset + type_size <= BLOCK_SZ);
        // 表示我们的Cache是一个脏的数据，将modified这一位置记为true
        self.modified = true;
        let addr = self.addr_of_offset(offset);
        unsafe { &mut *(addr as *mut T) }
    }
```
当缓存的生命周期结束后：需要写回脏的`Cache`。
```rust
impl Drop for BlockCache {
    fn drop(&mut self) {
        if self.modified {
            self.modified = false;
            self.block_device.write_block(self.block_id, &self.cache);
        }
    }
}
```
### 第二层的管理者：块缓存全局管理器
这个东东我们在上面源码分析中看过了，我现在不讲了。
## 第三层：磁盘层：超级块-索引块-数据块
### easy-fs 磁盘布局概述
easy-fs 磁盘按照块编号从小到大顺序分成 5 个连续区域：

- 第一个区域只包括一个块，它是 超级块 (Super Block)，用于定位其他连续区域的位置，检查文件系统合法性。

- 第二个区域是一个索引节点位图，长度为若干个块。它记录了索引节点区域中有哪些索引节点已经被分配出去使用了。

- 第三个区域是索引节点区域，长度为若干个块。其中的每个块都存储了若干个索引节点。

- 第四个区域是一个数据块位图，长度为若干个块。它记录了后面的数据块区域中有哪些已经被分配出去使用了。

- 最后的区域则是数据块区域，其中的每个被分配出去的块保存了文件或目录的具体内容。
### SuperBlock
已经得到分析，略过
### BitMap
位图用于管理索引结点和数据块，每个位图都由若干个块组成，每个块大小 4096 bits，每个 bit 都代表一个索引节点/数据块的分配状态。
```rust
/// A bitmap block
type BitmapBlock = [u64; 64];
/// Number of bits in a block
const BLOCK_BITS: usize = BLOCK_SZ * 8;
/// A bitmap
pub struct Bitmap {
    start_block_id: usize,
    blocks: usize,
}
```
`Bitmap`内存放着位图区域的起始块编号和块数，`BitmapBlock`将位图区域中的一个磁盘块解释为一个大数组，计算后发现与块大小是一致的，皆为 512 Bytes。

`Bitmap`负责了一定的分配事宜，我们来查看这是如何做到的。
```rust
impl Bitmap {
    /// 简单的初始化操作，没有什么好讲的。
    /// A new bitmap from start block id and number of blocks
    pub fn new(start_block_id: usize, blocks: usize) -> Self {
        Self {
            start_block_id,
            blocks,
        }
    }
    /// Allocate a new block from a block device
    /// blocks表示块的总数，一个块设备似乎是可以对应多个块的。
    pub fn alloc(&self, block_device: &Arc<dyn BlockDevice>) -> Option<usize> {
        for block_id in 0..self.blocks {
            let pos = get_block_cache(
                block_id + self.start_block_id as usize,
                Arc::clone(block_device),
            )
            .lock()
            .modify(0, |bitmap_block: &mut BitmapBlock| {
                // 遍历，去寻找trailing_ones尚没有满的东东，以此找到相关的块
                if let Some((bits64_pos, inner_pos)) = bitmap_block
                    .iter()
                    .enumerate()
                    .find(|(_, bits64)| **bits64 != u64::MAX)
                    .map(|(bits64_pos, bits64)| (bits64_pos, bits64.trailing_ones() as usize))
                {
                    // modify cache
                    bitmap_block[bits64_pos] |= 1u64 << inner_pos;
                    // 返回我们分配的具体位置，块-64bits单元-比特位置
                    // 通过遍历后找到一个后就马上返回了
                    Some(block_id * BLOCK_BITS + bits64_pos * 64 + inner_pos as usize)
                } else {
                    None
                }
            });
            if pos.is_some() {
                return pos;
            }
        }
        None
    }
    /// Deallocate a block
    /// 释放的方法是很类似的
    pub fn dealloc(&self, block_device: &Arc<dyn BlockDevice>, bit: usize) {
        let (block_pos, bits64_pos, inner_pos) = decomposition(bit);
        get_block_cache(block_pos + self.start_block_id, Arc::clone(block_device))
            .lock()
            .modify(0, |bitmap_block: &mut BitmapBlock| {
                assert!(bitmap_block[bits64_pos] & (1u64 << inner_pos) > 0);
                bitmap_block[bits64_pos] -= 1u64 << inner_pos;
            });
    }
    /// Get the max number of allocatable blocks
    pub fn maximum(&self) -> usize {
        self.blocks * BLOCK_BITS
    }
}
```
### 磁盘上的索引节点
这边的`DiskInodeType`其实就反映了两个经典的索引类型，其中一个是文件，另一个是目录。`direct`和`indirect1`与`indirect2`反映了这两类文件的访问方式。
```rust
/// A disk inode
#[repr(C)]
pub struct DiskInode {
    pub size: u32,
    pub direct: [u32; INODE_DIRECT_COUNT],
    pub indirect1: u32,
    pub indirect2: u32,
    type_: DiskInodeType,
}

#[derive(PartialEq)]
pub enum DiskInodeType {
    File,
    Directory,
}
```
文件如果较小，可以采用`direct`方式，最多可以链接`28`个数据块，而每个块大小为512 Byte，也就是说采用直接索引的方式文件最大可以是14 KB。

`indirect1`指向的是一个一级索引块，这个块可以链出`128`个用`u32`存储的数据块，这里就有64 KB了。在采用这一种链接的同时，`direct`部分也得塞满，

如果超过了78 KB的文件大小范围，则采用二级链接的方式来做。二级索引块可以链出`128`个一级索引块，对应的话就有8 MB这么大了。

在这边有一些简单的操作，我们分析如下：
```rust
    /// Return block number correspond to size.
    pub fn data_blocks(&self) -> u32 {
        Self::_data_blocks(self.size)
    }
    fn _data_blocks(size: u32) -> u32 {
        (size + BLOCK_SZ as u32 - 1) / BLOCK_SZ as u32
    }
    /// Return number of blocks needed include indirect1/2.
    pub fn total_blocks(size: u32) -> u32 {
        let data_blocks = Self::_data_blocks(size) as usize;
        let mut total = data_blocks as usize;
        // indirect1
        if data_blocks > INODE_DIRECT_COUNT {
            total += 1;
        }
        // indirect2
        if data_blocks > INDIRECT1_BOUND {
            total += 1;
            // sub indirect1，当然其中一个sub indirect1在上面的if else之中已经被添加上了，所以这边是从0开始count的
            total +=
                (data_blocks - INDIRECT1_BOUND + INODE_INDIRECT1_COUNT - 1) / INODE_INDIRECT1_COUNT;
        }
        total as u32
    }
```
块号取决于当前块的大小，由于需要额外的块来存储间接的索引，在这边需要简单地加上一些小数据。
```rust
    pub fn get_block_id(&self, inner_id: u32, block_device: &Arc<dyn BlockDevice>) -> u32 {
        let inner_id = inner_id as usize;
        if inner_id < INODE_DIRECT_COUNT {
            self.direct[inner_id]
        } else if inner_id < INDIRECT1_BOUND {
            get_block_cache(self.indirect1 as usize, Arc::clone(block_device))
                .lock()
                .read(0, |indirect_block: &IndirectBlock| {
                    indirect_block[inner_id - INODE_DIRECT_COUNT]
                })
        } else {
            let last = inner_id - INDIRECT1_BOUND;
            let indirect1 = get_block_cache(self.indirect2 as usize, Arc::clone(block_device))
                .lock()
                .read(0, |indirect2: &IndirectBlock| {
                    indirect2[last / INODE_INDIRECT1_COUNT]
                });
            get_block_cache(indirect1 as usize, Arc::clone(block_device))
                .lock()
                .read(0, |indirect1: &IndirectBlock| {
                    indirect1[last % INODE_INDIRECT1_COUNT]
                })
        }
    }
```
上面的这个函数很好地体现了逐级进行查找与索引的过程，请读者仔细理解这里设计的一些参数，考虑它们的意义是很有意思的。
```rust
    /// Inncrease the size of current disk inode
    pub fn increase_size(
        &mut self,
        new_size: u32,
        new_blocks: Vec<u32>,
        block_device: &Arc<dyn BlockDevice>,
    ) {
        let mut current_blocks = self.data_blocks();
        self.size = new_size;
        // 当size改变，对应所需的blocks也会改变
        let mut total_blocks = self.data_blocks();
        let mut new_blocks = new_blocks.into_iter();
        // fill direct，用新的blocks对应的地址进行填充，先填充直连的
        while current_blocks < total_blocks.min(INODE_DIRECT_COUNT as u32) {
            self.direct[current_blocks as usize] = new_blocks.next().unwrap();
            current_blocks += 1;
        }
        // alloc indirect1，刚刚填充了直连的，现在填充不直连的indirect1，但这个可能需要alloc了
        if total_blocks > INODE_DIRECT_COUNT as u32 {
            if current_blocks == INODE_DIRECT_COUNT as u32 {
                self.indirect1 = new_blocks.next().unwrap();
            }
            current_blocks -= INODE_DIRECT_COUNT as u32;
            total_blocks -= INODE_DIRECT_COUNT as u32;
        } else {
            return;
        }
        // fill indirect1
        get_block_cache(self.indirect1 as usize, Arc::clone(block_device))
            .lock()
            .modify(0, |indirect1: &mut IndirectBlock| {
                while current_blocks < total_blocks.min(INODE_INDIRECT1_COUNT as u32) {
                    indirect1[current_blocks as usize] = new_blocks.next().unwrap();
                    current_blocks += 1;
                }
            });
        // alloc indirect2
        // 先前已经减去过INODE_DIRECT_COUNT了
        if total_blocks > INODE_INDIRECT1_COUNT as u32 {
            if current_blocks == INODE_INDIRECT1_COUNT as u32 {
                self.indirect2 = new_blocks.next().unwrap();
            }
            current_blocks -= INODE_INDIRECT1_COUNT as u32;
            total_blocks -= INODE_INDIRECT1_COUNT as u32;
        } else {
            return;
        }
        // fill indirect2 from (a0, b0) -> (a1, b1)
        let mut a0 = current_blocks as usize / INODE_INDIRECT1_COUNT;
        let mut b0 = current_blocks as usize % INODE_INDIRECT1_COUNT;
        let a1 = total_blocks as usize / INODE_INDIRECT1_COUNT;
        let b1 = total_blocks as usize % INODE_INDIRECT1_COUNT;
        // alloc low-level indirect1
        get_block_cache(self.indirect2 as usize, Arc::clone(block_device))
            .lock()
            .modify(0, |indirect2: &mut IndirectBlock| {
                while (a0 < a1) || (a0 == a1 && b0 < b1) {
                    if b0 == 0 {
                        indirect2[a0] = new_blocks.next().unwrap();
                    }
                    // fill current
                    get_block_cache(indirect2[a0] as usize, Arc::clone(block_device))
                        .lock()
                        .modify(0, |indirect1: &mut IndirectBlock| {
                            indirect1[b0] = new_blocks.next().unwrap();
                        });
                    // move to next
                    b0 += 1;
                    if b0 == INODE_INDIRECT1_COUNT {
                        b0 = 0;
                        a0 += 1;
                    }
                }
            });
    }
```
本质上其实还是逐个等级进行遍历寻找的一个漫长的过程，这很酷。
```rust
    /// Clear size to zero and return blocks that should be deallocated.
    /// We will clear the block contents to zero later.
    pub fn clear_size(&mut self, block_device: &Arc<dyn BlockDevice>) -> Vec<u32> {
        let mut v: Vec<u32> = Vec::new();
        let mut data_blocks = self.data_blocks() as usize;
        self.size = 0;
        let mut current_blocks = 0usize;
        // direct，如果文件确实很小，我们直接清空direct位置就好了。
        while current_blocks < data_blocks.min(INODE_DIRECT_COUNT) {
            v.push(self.direct[current_blocks]);
            self.direct[current_blocks] = 0;
            current_blocks += 1;
        }
        // indirect1 block
        if data_blocks > INODE_DIRECT_COUNT {
            v.push(self.indirect1);
            data_blocks -= INODE_DIRECT_COUNT;
            current_blocks = 0;
        } else {
            return v;
        }
        // indirect1
        get_block_cache(self.indirect1 as usize, Arc::clone(block_device))
            .lock()
            .modify(0, |indirect1: &mut IndirectBlock| {
                while current_blocks < data_blocks.min(INODE_INDIRECT1_COUNT) {
                    v.push(indirect1[current_blocks]);
                    //indirect1[current_blocks] = 0;
                    // 这边的是可选的，可以清空它，当然也可以不清空
                    // 原因很简单：因为本质上indirect1和indirect2实际上是一个索引，而不是其他的什么东西，我并不
                    // 一定要把block进行清空的操作，可能只需要把索引置为0就行了。
                    current_blocks += 1;
                }
            });
        self.indirect1 = 0;
        // indirect2 block
        if data_blocks > INODE_INDIRECT1_COUNT {
            v.push(self.indirect2);
            data_blocks -= INODE_INDIRECT1_COUNT;
        } else {
            return v;
        }
        // indirect2
        assert!(data_blocks <= INODE_INDIRECT2_COUNT);
        let a1 = data_blocks / INODE_INDIRECT1_COUNT;
        let b1 = data_blocks % INODE_INDIRECT1_COUNT;
        get_block_cache(self.indirect2 as usize, Arc::clone(block_device))
            .lock()
            .modify(0, |indirect2: &mut IndirectBlock| {
                // full indirect1 blocks
                for entry in indirect2.iter_mut().take(a1) {
                    v.push(*entry);
                    get_block_cache(*entry as usize, Arc::clone(block_device))
                        .lock()
                        .modify(0, |indirect1: &mut IndirectBlock| {
                            for entry in indirect1.iter() {
                                v.push(*entry);
                            }
                        });
                }
                // last indirect1 block，这边所示的应该是不完整的indirect2索引，因此所用的方式实际上是有点不一样的。
                // 前面需要全部进行遍历，但是这边不需要，这边遍历到b1就可以了。
                if b1 > 0 {
                    v.push(indirect2[a1]);
                    get_block_cache(indirect2[a1] as usize, Arc::clone(block_device))
                        .lock()
                        .modify(0, |indirect1: &mut IndirectBlock| {
                            for entry in indirect1.iter().take(b1) {
                                v.push(*entry);
                            }
                        });
                    //indirect2[a1] = 0;
                }
            });
        self.indirect2 = 0;
        v
    }
```
从上面的注释分析中发现其实这段代码只是长而已，阅读起来其实没有太大的压力~

我们再来看函数`read_at`，这个函数将会从当前的磁盘索引点位置读取相关的数据。
```rust
    /// Read data from current disk inode
    pub fn read_at(
        &self,
        offset: usize,
        buf: &mut [u8],
        block_device: &Arc<dyn BlockDevice>,
    ) -> usize {
        // 起始点，当然start是一个动态变化的
        let mut start = offset;
        // 根据buf数组的大小决定接下来一共阅读多少位，总而言之，最后的东西将会被读到buf之中
        let end = (offset + buf.len()).min(self.size as usize);
        if start >= end {
            return 0;
        }
        // 起始位置块
        let mut start_block = start / BLOCK_SZ;
        let mut read_size = 0usize;
        // 体现出了一个逐个块进行阅读处理的基本流程，利用start作为cursor索引者，一个一个地阅读
        loop {
            // calculate end of current block
            let mut end_current_block = (start / BLOCK_SZ + 1) * BLOCK_SZ;
            end_current_block = end_current_block.min(end);
            // read and update read size
            let block_read_size = end_current_block - start;
            let dst = &mut buf[read_size..read_size + block_read_size];
            get_block_cache(
                self.get_block_id(start_block as u32, block_device) as usize,
                Arc::clone(block_device),
            )
            .lock()
            .read(0, |data_block: &DataBlock| {
                // 非常强暴地直接复制粘贴
                let src = &data_block[start % BLOCK_SZ..start % BLOCK_SZ + block_read_size];
                dst.copy_from_slice(src);
            });
            read_size += block_read_size;
            // move to next block
            if end_current_block == end {
                break;
            }
            start_block += 1;
            start = end_current_block;
        }
        read_size
    }
```
`write_at`是一个反向操作，从`buf`之中读取到相关的块中，我们在这边就略过吧，不再赘述了。

### 目录项：
其实本质上写的还是比较浅显的，随便看一下吧。
```rust
/// Size of a directory entry
pub const DIRENT_SZ: usize = 32;

impl DirEntry {
    /// Create an empty directory entry
    pub fn empty() -> Self {
        Self {
            name: [0u8; NAME_LENGTH_LIMIT + 1],
            inode_id: 0,
        }
    }
    /// Crate a directory entry from name and inode number
    pub fn new(name: &str, inode_id: u32) -> Self {
        let mut bytes = [0u8; NAME_LENGTH_LIMIT + 1];
        bytes[..name.len()].copy_from_slice(name.as_bytes());
        Self {
            name: bytes,
            inode_id,
        }
    }
    /// Serialize into bytes
    pub fn as_bytes(&self) -> &[u8] {
        unsafe { core::slice::from_raw_parts(self as *const _ as usize as *const u8, DIRENT_SZ) }
    }
    /// Serialize into mutable bytes
    pub fn as_bytes_mut(&mut self) -> &mut [u8] {
        unsafe { core::slice::from_raw_parts_mut(self as *mut _ as usize as *mut u8, DIRENT_SZ) }
    }
    /// Get name of the entry
    pub fn name(&self) -> &str {
        // 切片确实存在这个麻烦：我需要通过*i来找到\0字符所在的位置，以选择真正的目录名字
        let len = (0usize..).find(|i| self.name[*i] == 0).unwrap();
        core::str::from_utf8(&self.name[..len]).unwrap()
    }
    /// Get inode number of the entry
    pub fn inode_id(&self) -> u32 {
        self.inode_id
    }
}
```
## 第四层：磁盘块的管理器
文件系统结构如下所示，我们已经在前面对此有所提及：
```rust
///An easy file system on block
pub struct EasyFileSystem {
    ///Real device
    pub block_device: Arc<dyn BlockDevice>,
    ///Inode bitmap
    pub inode_bitmap: Bitmap,
    ///Data bitmap
    pub data_bitmap: Bitmap,
    inode_area_start_block: u32,
    data_area_start_block: u32,
}
```
如何创建一个文件系统以供操作系统使用？我们具体来看一下`create`函数的代码：
```rust
    pub fn create(
        block_device: Arc<dyn BlockDevice>,
        total_blocks: u32,
        inode_bitmap_blocks: u32,
    ) -> Arc<Mutex<Self>> {
        // calculate block size of areas & create bitmaps
        // 文件系统需要有inode_bitmap，
        // 需要有blocks块号和blocks数目以完成初始化
        
        /// 请回想我们之前在easy-fs 磁盘布局中的一些描述
        /// 一、超级块部分，一共就一个那么大。
        /// 二、inode_bitmap位图部分，其中的new两个参数一个是块起始位置，一个是一共有多少个块来存放位图信息，占据inode_bitmap_blocks这么多个块
        /// 三、inode对应的索引结点区域，由很多个DiskInode组成的，具体有多少个DiskInode取决于inode_bitmap_blocks有多少块
        /// 当然，有多少个DiskInode -> 要用多少个块来存储之？这就是第三个部分
        let inode_bitmap = Bitmap::new(1, inode_bitmap_blocks as usize);
        /// bitmap之中，每个bit位均表示一个DiskInode空间。心在一共有inode_bitmap_blocks这么多个块，每个块有512比特，就有4096个 bit位
        /// 通过相乘，得到一共的DiskInode要占多大的空间
        let inode_num = inode_bitmap.maximum();
        let inode_area_blocks =
            ((inode_num * core::mem::size_of::<DiskInode>() + BLOCK_SZ - 1) / BLOCK_SZ) as u32;
        let inode_total_blocks = inode_bitmap_blocks + inode_area_blocks;
        /// 需要减去一个superblock，以及inode_total_blocks
        let data_total_blocks = total_blocks - 1 - inode_total_blocks;
        /// 我们希望数据块位图中的每个bit仍然能够对应到一个数据块，
        /// 但是数据块位图又不能过小，不然会造成某些数据块永远不会被使用。
        /// 因此数据块位图区域最合理的大小是剩余的块数除以 4097 再上取整，
        /// 因为位图中的每个块能够对应 4096 个数据块。其余的块就都作为数据块使用。
        /// data_bitmap上的对应关系与前面的DiskInode对应关系有所不同
        let data_bitmap_blocks = (data_total_blocks + 4096) / 4097;
        let data_area_blocks = data_total_blocks - data_bitmap_blocks;
        let data_bitmap = Bitmap::new(
            (1 + inode_bitmap_blocks + inode_area_blocks) as usize,
            data_bitmap_blocks as usize,
        );
        /// 文件系统的创立
        let mut efs = Self {
            block_device: Arc::clone(&block_device),
            inode_bitmap,
            data_bitmap,
            inode_area_start_block: 1 + inode_bitmap_blocks,
            data_area_start_block: 1 + inode_total_blocks + data_bitmap_blocks,
        };
        // clear all blocks
        for i in 0..total_blocks {
            get_block_cache(i as usize, Arc::clone(&block_device))
                .lock()
                /// 为了清空块，把每个Block均以DataBlock的方式进行解读，这个Block对应的是一个[u8; 512]的数组
                .modify(0, |data_block: &mut DataBlock| {
                    for byte in data_block.iter_mut() {
                        *byte = 0;
                    }
                });
        }
        // initialize SuperBlock
        /// 对第0个块进行初始化，初始化的方式是super_block型
        /// 特别注意这里加上了魔数，magic number
        get_block_cache(0, Arc::clone(&block_device)).lock().modify(
            0,
            |super_block: &mut SuperBlock| {
                super_block.initialize(
                    total_blocks,
                    inode_bitmap_blocks,
                    inode_area_blocks,
                    data_bitmap_blocks,
                    data_area_blocks,
                );
            },
        );
        // write back immediately
        // create a inode for root node "/"
        /// 为根结点分配一个索引结点，以方便我们后续的索引工作
        /// 索引的第一个位置，即0，但是其block位置不在0，而是inode_area的起始位置
        assert_eq!(efs.alloc_inode(), 0);
        /// 根据efs中的block_id位置确定对应的block_id和offset，当然是从inode_area位置起始的
        let (root_inode_block_id, root_inode_offset) = efs.get_disk_inode_pos(0);
        get_block_cache(root_inode_block_id as usize, Arc::clone(&block_device))
            .lock()
            .modify(root_inode_offset, |disk_inode: &mut DiskInode| {
                /// 我们分配好了空间，也知道了它对应的root_inode_block_id即具体位置，现在要对这个索引结点，即DiskInode进行初始化
                disk_inode.initialize(DiskInodeType::Directory);
            });
        /// 要把cache中的脏内容写回内存block，让大家都知道有这么一回事了。这个具体实现并不在库函数里
        block_cache_sync_all();
        Arc::new(Mutex::new(efs))
    }
```
在利用了`Create`描述好磁盘的布局后，我们可以采用`open`来读取这个信息，我们不再赘述了，因为之前确实分析过。
## 索引结点：
`efs`的文件系统已经描述完毕，但是为了其易用性，需要做一层`vfs`的封装，因为文件系统的使用者并不关心具体的磁盘分布是怎么样的，因此`vfs`的设置如下所示：
```rust
/// Virtual filesystem layer over easy-fs
pub struct Inode {
    block_id: usize,
    block_offset: usize,
    fs: Arc<Mutex<EasyFileSystem>>,
    block_device: Arc<dyn BlockDevice>,
}
```
这个数据结构直接把`filesystem`封装好了，作为第三个参数。`block_id`和`block_offset`表示的是这个`Inode`信息对应的根结点`DiskInode`在磁盘上的具体存储位置，以方便使用者对其进行一些修改。

我们再次强调一下：`Inode`是文件系统使用者的视角，根据`block_id`和`block_offset`可以找到其所对应的`DiskInode`结构，也就是说有很多个`Inode`和`DiskInode`，具体怎么对应上的，请看`Create`函数。
```rust
    /// 简化访问流程，避免一些复杂的问题
    /// Call a function over a disk inode to read it
    fn read_disk_inode<V>(&self, f: impl FnOnce(&DiskInode) -> V) -> V {
        get_block_cache(self.block_id, Arc::clone(&self.block_device))
            .lock()
            .read(self.block_offset, f)
    }
    /// Call a function over a disk inode to modify it
    fn modify_disk_inode<V>(&self, f: impl FnOnce(&mut DiskInode) -> V) -> V {
        get_block_cache(self.block_id, Arc::clone(&self.block_device))
            .lock()
            .modify(self.block_offset, f)
    }
```
### 获取根目录的 inode
这个是`efs`中提供的功能：本质上是获取`0`号`inode`，即`root_inode`所存在的具体位置。
```rust
    /// Get the root inode of the filesystem
    pub fn root_inode(efs: &Arc<Mutex<Self>>) -> Inode {
        let block_device = Arc::clone(&efs.lock().block_device);
        // acquire efs lock temporarily
        let (block_id, block_offset) = efs.lock().get_disk_inode_pos(0);
        // release efs lock
        Inode::new(block_id, block_offset, Arc::clone(efs), block_device)
    }
```
### 文件索引
所有的文件都在`root_inode`之下，这也影响了我们查找文件的方式：
```rust
    pub fn find(&self, name: &str) -> Option<Arc<Inode>> {
        let fs = self.fs.lock();
        /// 利用read_disk_inode找到根目录inode结点，并以此为基准操作
        self.read_disk_inode(|disk_inode| {
            /// 根据name找到全部对应的DiskInode的inode_id值
            self.find_inode_id(name, disk_inode)
            .map(|inode_id| {
                /// 根据inode_id找到对应的block_id和block_offset位置
                let (block_id, block_offset) = fs.get_disk_inode_pos(inode_id);
                // 返回Inode
                Arc::new(Self::new(
                    block_id,
                    block_offset,
                    self.fs.clone(),
                    self.block_device.clone(),
                ))
            })
        })
    }

    /// Find inode under a disk inode by name
    fn find_inode_id(&self, name: &str, disk_inode: &DiskInode) -> Option<u32> {
        // assert it is a directory
        assert!(disk_inode.is_dir());
        // 利用根目录结点上所写的size大小来得知一共有多少个文件/目录
        let file_count = (disk_inode.size as usize) / DIRENT_SZ;
        let mut dirent = DirEntry::empty();
        for i in 0..file_count {
            assert_eq!(
                // 从offset上的修改来遍历目录下的文件/目录，读取dirent内的长度为DIRENT_SZ信息，里头主要是名字
                // 查看DiskInode数据结构就能搞懂了
                disk_inode.read_at(DIRENT_SZ * i, dirent.as_bytes_mut(), &self.block_device,),
                DIRENT_SZ,
            );
            // 判断名字是否相等，相等则返回之
            if dirent.name() == name {
                return Some(dirent.inode_id() as u32);
            }
        }
        None
    }
```
### 文件列举
如法炮制
### 文件创建
为什么上面的方式可以读取名字？具体来说，得看一下相关的`disk_inode`是怎么被创建出来的，也得看`disk_inode`和`inode`之间的对应关系是怎么建立起来的：
```rust
    /// Create inode under current inode by name
    pub fn create(&self, name: &str) -> Option<Arc<Inode>> {
        let mut fs = self.fs.lock();
        // 设置op为一个函数闭包
        let op = |root_inode: &DiskInode| {
            // assert it is a directory
            assert!(root_inode.is_dir());
            // has the file been created?
            self.find_inode_id(name, root_inode)
        };
        // 先看root_inode对应的inode群中是否有为name名字的文件或者目录
        // 如果找到了，那显然不行的，要退出
        if self.read_disk_inode(op).is_some() {
            return None;
        }
        // create a new file
        // alloc a inode with an indirect block
        // 在文件系统重分配一个new_inode_id，并对其进行初始化
        let new_inode_id = fs.alloc_inode();
        // initialize inode
        let (new_inode_block_id, new_inode_block_offset) = fs.get_disk_inode_pos(new_inode_id);
        get_block_cache(new_inode_block_id as usize, Arc::clone(&self.block_device))
            .lock()
            .modify(new_inode_block_offset, |new_inode: &mut DiskInode| {
                new_inode.initialize(DiskInodeType::File);
            });

        // 初始化第二步，对结点进行修改，写入相关的文件信息
        // 当然，是在当前的这个inode之下
        self.modify_disk_inode(|root_inode| {
            // append file in the dirent
            // 先看看当前有多少个file
            let file_count = (root_inode.size as usize) / DIRENT_SZ;
            let new_size = (file_count + 1) * DIRENT_SZ;
            // increase size
            self.increase_size(new_size as u32, root_inode, &mut fs);
            // write dirent
            let dirent = DirEntry::new(name, new_inode_id);
            // 把新的数据写入到对应的inode的位置
            root_inode.write_at(
                file_count * DIRENT_SZ,
                dirent.as_bytes(),
                &self.block_device,
            );
        });

        let (block_id, block_offset) = fs.get_disk_inode_pos(new_inode_id);
        block_cache_sync_all();
        // return inode
        Some(Arc::new(Self::new(
            block_id,
            block_offset,
            self.fs.clone(),
            self.block_device.clone(),
        )))
        // release efs lock automatically by compiler
    }
```