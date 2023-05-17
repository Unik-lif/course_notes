## 第六章：文件系统
`easyfs`文件系统的框架：
1. 磁盘块设备接口层：读写磁盘块设备的`trait`接口
2. 块缓存层：位于内存的磁盘块数据缓存
3. 磁盘数据结构层：表示磁盘文件系统的数据结构
4. 磁盘块管理层：实现对磁盘文件系统的管理
5. 索引节点层：实现文件创建，文件打开，文件读写等操作

### easyfs系统
这个系统似乎是以库函数的方式管理的。

## 读源码：
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