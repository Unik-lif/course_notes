刚刚从北京回家，很累，今天开摆一天，简单看一看交大的`PPT`学习一下。
## lec1: intro
Therac-25: a radiation therapy machine, injured patients died from lethal dosage of radiation caused by a software bug.

### system & complexity
system: interacting set of components with a specified behavior at the interface with its environment

a systematical way to understand a complex system is needed.

we get 14 properites of computer systems:
1. correctness
- every third bug is not a bug, it's not a bug, it's a feature.
- how to describe the functionalities of your code?
2. latency
3. throughput
4. scalability
5. utilization
6. performance isolation
- see 'Perfiso: performance isolation for commercial latency-sensitive services'.
7. energy efficientcy
8. consistency
9. fault tolerance
- cosmic radiation bug.
- rowhammer attack.
10. security
- cold-boot attack
11. privacy
- privacy mode is not so powerful as described.
12. trust
13. compatibility
- itanium, not compatible to x86, died.
14. usability
- iphone

These properties have some confflicts. => all these lead to more complexity.

Problems:
1. Emergent properties => properties are not considered at design time.
- speed up of Ethernet, some changes to detect collision.
2. Propagation of effects => butterfly effort.
3. Incommensurate scaling => design for small model may not scale.
- different parts of the system exhibit different orders of growth.
4. Trade-offs => can't sell the cow and drink the milk.
### Handle Complexity:
cases studies.

systems are indefinitely: Limits the depth of digging.

Some methods:
1. Modularity: split up system
2. Abstraction: Interface/Hiding
3. Layering: gradually build up capabilities
4. Hierarchy: Reduce connections, divide & conquer
### Paper reading:
Perfiso: performance isolation for commercial latency-sensitive services
## lec2: Scalability in Practice
In modern Life, each click on the website needs thousands of servers to cooperate.

Build a website on one machine:
```
https://www.taobao.com => Internet ====> Application ==> Database: user, price
                                              |
                                              | =======> File: Image
```
Limitations:
1. The disk & memory of one server => can't store massive amount of data.
2. Applications uses the CPU for processing, while a single CPU can hardly scale due to the end of **Moore's Law & Dennard Scaling**.
- Dennard scaling: as transistors get smaller, their power density stays constant.

To scale:
1. disaggregating application & data.
- more CPUs to process application.
- more disk & cache to maintain database.
- more disks to store large bulk of data for the File system
2. Caching: avoid slow data accesses.
- distributed caching server.
- [consistent hasing to find the server to cache data.](https://www.bilibili.com/video/BV1q4411w7Dw?p=2&vd_source=49f5b184846e52adee8a1a5165c5f962)
3. more servers: for stateless application servers.
- only executes the logic relies on input data.
- stateless servers can have better fault-tolerance and better elasticity.
4. scaling database
- separate the database for read/write
- separate a table on multiple databases
5. using distributed file system
6. using CDN.
- content delivery network caches the content at the network providers, which is closer to users.
7. separate different applications + distributed computing
- dedicated servers for different applications.

Distributed System: a collection of independent/autonomous connected through a communication network working together hosts to perform a service.

### Faults:
faults are common especially in distributed systems => Scale!

MTBF: mean time between failures

in a distributed system, some parts of the system can be broken in some unpredictable way, such failure is partial (aka. grey failure)

We want the system still working!

common fault: network partition
### reliability:
Metrics to measure reliability:
- MTTF: mean time to failure
- MTTR: mean time to repair
- MTBF: mean time between failure
- MTBF = MTTF + MTTR

on the other hand, large-scale systems can be highly available.

challenge: consistency

The CAP theorem:
- consistency, availability & partition tolerance
- C: all nodes see the same data at the same time
- A: a guarantee that every request receives a response about whether it succeeded or failed.
- P: the system continues to operate despite arbitrary message loss or failure of part of the system.

AP: sacrifice C.

CP: sacrifice A.
### Paper reading:
The Datacenter as a Computer - An introduction to the design of warehouse scale machines.

## Lec3: Inode-based File System
Two key properties:
1. durable & has a name.
2. high-level version of the memory abstraction

The big picture is shown below:
```
User: 
    App-1 App-2 App-3
--------------------------    Open("a.txt", "rw")
                              Read(...)
Kernel:                       Write(...)
        File System           
        Disk Driver

--------------------------    Read(block_addr, buf)
Hardware:                     Write(block_addr, buf)
    Memory          Disk
```
In Unix File System, many APIs <=> Abstractions exist.

In a Naive File System, each file occupies one continuous range of blocks.
- Use `block index` as file name.
- Every file write will either `append` or `reallocate`

The problems exist: Fragmentation.

### inode: 7 software layers.
#### L1: block layer
This layer **maps the block number to the block data**.
```C
procedure: block_number_to_block(int b) -> block

return devices[b]
```
How to know the size of block?
- metadata => super block!

In a file system, we got one superblock, kernel reads superblock when mount the FS.

it contains:
- size of the blocks: always a trade-offs
- number of free blocks
- a list of free blocks: to track free blocks, we can use a bitmap.
- other metadata of the file system (including inode info)
#### L2: file layer
File requirements:
- store items that are larger than one block
- may grow or shrink over time
- a file is a linear array of bytes of arbitrary length
- record which blocks belong to each file

inode(index node)
- a container for metadata about the file.
```C
struct inode {
    int block_nums[N];
    int size;
};
```

File -> Block -> Disk Block
```C
procedure inode_to_block(int offset, inode i) -> block
    o <- offset/BLOCKSIZE
    b = index_to_block_num(i, o)
    return block_number_to_block(b)

procedure index_to_block_number(inode i, int index) -> int
    return i.block_nums[index]
```
#### L3: inode number layer
mapping: inode number -> inode
```C
procedure inode_number_to_inode(int num) -> inode
    return inode_table[num]
```

managers the inode using a inode table.
```C
procedure inode_number_to_block(int offset, int inode_number) -> block
    inode i = inode_number_to_inode(inode_number)
    o <- offset/BLOCKSIZE
    b <- index_to_block_number(i, o)
    return block_number_to_block(b)
```
Now the OS get a way to use inode to operate on the device blocks, but it is not user-friendly, we should provide another abstraction layer to mke it more user-friendly.

#### L4: File Name Layer
File name
- Hide metadata of file management
- Files and I/O devices

Mapping:
- mapping table is saved in directory
- default context: current working directory
- - context reference is an inode number
- - the current working directory is also a file

```C
procedure name_to_inode(string filename, int dir) -> int
    return lookup(dir, filename)

procedure lookup(string filenmae, int dir) -> int
    block b
    inode i = inode_number_to_inode(dir)
    if i.type!= Directory then return failure
    for offset from 0 to i.size - 1 do
        b <- inode_number_to_block(offset, dir)
        if string_match(filename, b) then
            return inode_number(filename, b)
        offset <- offset + BLOCKSIZE
    return failure
```
#### L5: path name layer
Also describes the directories and files.

links: create shortcut for long names.

unlink: remove the binding of filename to inode number.

a inode can bind multiple file names.

No cycle for link => cause very strange results.

#### L6: absolute path name layer
#### L7: symbolic link layer
Name files on other disks
- inode is different on other disks
- supports to attach new disks to the name space

soft link(symbolic link) -> add another type of indoe

soft-link links file by file name.

soft-link: to use the physical directory structure instead of following symbolic links.
```
cd -P ..
```
## Lec4: File system API and Disk I/O
File System API is implemented as system calls to user applications.

### Why File Descriptor?
Other options:
- Option-1: OS returns an inode pointer
- Option-2: OS returns all the block numbers of the file

Reasons:
- user can never access kernel's data structure
- all file operationsa are done by the kernel

fd_table vs file_table
- one file_table for the whole system: records info for opened files
- one fd_table for each process: records mapping of fd to index of the file_table

something like this:
![](pngs/Screenshot%202024-02-07%20094922.png)

Some API implementation Methodlogy:
- fetch the inode -> get basic info of the file.
- fetch the block -> get the context of the file.

for atime in inode => reflects the access time of the file corresponds to the inode.

The updates of the file system might be interrupted, so how can we make it atomic?
- a very good question to think about.

Other filesystems: FAT - file allocation tables.
### FAT32
use linked list to manage the blocks.