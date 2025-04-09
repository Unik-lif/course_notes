## 学习PPT
时隔一年半又过来学习了,主要是有文件系统的需求,需要搞清楚怎么做.

Filesystem Abstraction
- superblock: info about the filesystem instance such as the block size, the root inode, filesystem size. Present both on storage and in memory.
- file: info about an opened file such as the current file pointer, only exists in memory.
- inode: identifying a file on disk, exists both on storage and in memory.
- dentry: associates a name with an inode, exists both on storage and in memory.

VFS: virtual file system


