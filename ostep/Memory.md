## The Abstraction: Address Spaces
In early days, the OS Memory allocation is quite easy.
```
-------------------- 0 kB
|        OS        |
-------------------- 64 kB
|                  |
|  Current Program |
|                  |
-------------------- max
```
After a while, people began demanding more of machines, and the era of time sharing was born.

Simply storing process info and store-and-fetch them will be slow, so people prefer to give each task a part of memory and simply switch between them.
```
-------------------- 0 kB
|        OS        |
-------------------- 64 kB
|                  |
|      (free)      |
|                  |
-------------------- 128 kB
|    Process C     |
-------------------- 192 kB
|    Process B     |
-------------------- 256 kB
etc...
```
for every process, OS give them an illusion that running program thinks it is loaded into memory at a particular address and has a potentially very large address space.
```
-------------------- 0 kB
|   Program Code   |
-------------------- 1 kB
|       Heap       |
-------------------- 2 kB
|        |         |
|                  |
|      (free)      |
|        ^         |
|        |         |
-------------------- 15 kB
|       Stack      |
-------------------- 16 kB
```

How does OS do it?
### Virtualization
Goal: the OS should implement virtual memory in a way that is invisible to the running program.
1. **Transparency:** OS and Hardwares do all the work to multiplex memory among many different jobs.
2. **Efficiency:** OS should strive to make the virtualization as efficient as possible, both in terms of time and space.
3. **Protection:** OS should make sure to protect processes from one another as well as the OS itself from processes. Processes should also be isolated from each other.

### HW:
1. The RTFM is very interesting.
2. 7 GB. -> nearly 8 GB in fact.
3. √
4. Nop. Didn't match my expectations, but always in a confidence interval. A little big -> match. Too big -> segmentation fault.
5. pmap: process id -> map to -> files.
6. In short, many parts of VM.
above all: pmap utility gathers most of its information from the `/proc/PID/smaps` file and makes it friendly to humans.
```
95:   ./memory-user 1024 100
     Address Perm   Offset Device            Inode    Size     Rss     Pss Referenced Anonymous LazyFree ShmemPmdMapped FilePmdMapped Shared_Hugetlb Private_Hugetlb Swap SwapPss Locked THPeligible Mapping     55b0b348d000 r--p 00000000  00:3a 2533274790494850       4       4       4          4         0        0              0             0              0               0    0       0      0           0 memory-user 55b0b348e000 r-xp 00001000  00:3a 2533274790494850       4       4       4          4         0        0              0             0              0               0    0       0      0           0 memory-user 55b0b348f000 r--p 00002000  00:3a 2533274790494850       4       4       4          4         0        0              0             0              0               0    0       0      0           0 memory-user 55b0b3490000 r--p 00002000  00:3a 2533274790494850       4       4       4          4         4        0              0             0              0               0    0       0      0           0 memory-user 55b0b3491000 rw-p 00003000  00:3a 2533274790494850       4       4       4          4         4        0              0             0              0               0    0       0      0           0 memory-user 55b0b53b9000 rw-p 00000000  00:00                0     132       4       4          4         4        0              0             0              0               0    0       0      0           0 [heap]      7f2412a8a000 rw-p 00000000  00:00                0 1048592 1048588 1048588    1048588   1048588        0              0             0              0               0    0       0      0           1
7f2452a8e000 r--p 00000000  08:20             6499     160     160      22        160         0        0              0             0              0               0    0       0      0           0 libc.so.6   7f2452ab6000 r-xp 00028000  08:20             6499    1620     788     118        788         0        0              0             0              0               0    0       0      0           0 libc.so.6   7f2452c4b000 r--p 001bd000  08:20             6499     352      60       8         60         0        0              0             0              0               0    0       0      0           0 libc.so.6   7f2452ca3000 r--p 00214000  08:20             6499      16      16      16         16        16        0              0             0              0               0    0       0      0           0 libc.so.6   7f2452ca7000 rw-p 00218000  08:20             6499       8       8       8          8         8        0              0             0              0               0    0       0      0           0 libc.so.6   7f2452ca9000 rw-p 00000000  00:00                0      52      16      16         16        16        0              0             0              0               0    0       0      0           0
7f2452cc0000 rw-p 00000000  00:00                0       8       4       4          4         4        0              0             0              0               0    0       0      0           0
7f2452cc2000 r--p 00000000  08:20             6303       8       8       1          8         0        0              0             0              0               0    0       0      0           0 ld-linux-x86-64.so.2
7f2452cc4000 r-xp 00002000  08:20             6303     168     168      23        168         0        0              0             0              0               0    0       0      0           0 ld-linux-x86-64.so.2
7f2452cee000 r--p 0002c000  08:20             6303      44      40       6         40         0        0              0             0              0               0    0       0      0           0 ld-linux-x86-64.so.2
7f2452cfa000 r--p 00037000  08:20             6303       8       8       8          8         8        0              0             0              0               0    0       0      0           0 ld-linux-x86-64.so.2
7f2452cfc000 rw-p 00039000  08:20             6303       8       8       8          8         8        0              0             0              0               0    0       0      0           0 ld-linux-x86-64.so.2
7ffdf0eda000 rw-p 00000000  00:00                0     132      12      12         12        12        0              0             0              0               0    0       0      0           0 [stack]     7ffdf0f58000 r--p 00000000  00:00                0      16       0       0          0         0        0              0             0              0               0    0       0      0           0 [vvar]      7ffdf0f5c000 r-xp 00000000  00:00                0       8       4       0          4         0        0              0             0              0               0    0       0      0           0 [vdso]                                                         ======= ======= ======= ========== ========= ======== ============== ============= ============== =============== ==== ======= ====== ===========
                                                   1051352 1049912 1048862    1049912   1048672        0              0             0              0               0    0       0      0           1 KB
```
7. for pmap result of 1024 MB and 512 MB, the result is shown below:
```
00007facbea3a000 1048592K rw---   [ anon ]
00007facfea3e000    160K r---- libc.so.6

00007fbfcf786000 524304K rw---   [ anon ]
00007fbfef78a000    160K r---- libc.so.6
```
看起来似乎分配类似是在栈上？确实感觉有点诡异哦。
简单地做了几组数据，发现了一个现象：
```
linkvm@link:/mnt/d/OSTEP-homework/vm-api$ ./memory-user 1024 10 // malloc 1024个字节
malloc: 0x5619558f52a0
position = 0x7fe0ed7e3010
^C
linkvm@link:/mnt/d/OSTEP-homework/vm-api$ vim memory-user.c
linkvm@link:/mnt/d/OSTEP-homework/vm-api$ gcc -o memory-user memory-user.c
linkvm@link:/mnt/d/OSTEP-homework/vm-api$ ./memory-user 1024 10 // malloc 1 MB 个字节
malloc: 0x7f02a427a010
position = 0x7f02a437b010
^C
```
对于大分配，会倾向于放在栈上，对于小分配，会倾向于放在堆上。
## Interlude: Memory API
### Key question: How to allocate and manage memory.
#### sizeof function.
return 4 or 8 Bytes for pointer. return exactly how big is for arrays.

Convention for string space: malloc(strlen(s) + 1)

#### function strdup
```
# t is the duplicate of s.
linkvm@link:/mnt/d/OSTEP-homework/vm-api$ ./strdup
hallo t position: 0x558db40932a0 s position: 0x7ffe32265962
```
#### Dangling pointer:
Free the memory before it is finished using it. The subsequent use can crash the program.
### Memory Leakage:
For regular user, even though you don't use 'free' to release the heap, the OS will automatically do it for you when the process is finished. However for long-running server or database, this will be a great threat.

#### Some tools might be useful:
Valgrind & Purify.
#### Others:
'malloc' is a lib function based on 'brk' and 'sbrk' syscalls, and it is also a relatively portable API for users.

'mmap' call: map a file/device into the memory.

### HW:
1. Segmentation fault.
2. First of all, I can't set breakpoints, secondly, gdb will cause a SIGSEGV fault.
3. Valgrind detect where the fault it is.
```
 19 ==12747== Process terminating with default action of signal 11 (SIGSEGV)
 18 ==12747==  Access not within mapped region at address 0x0
 17 ==12747==    at 0x109151: printf (stdio2.h:112)
 16 ==12747==    by 0x109151: main (null.c:7)
 15 ==12747==  If you believe this happened as a result of a stack
 14 ==12747==  overflow in your program's main thread (unlikely but
 13 ==12747==  possible), you can try to increase the size of the
 12 ==12747==  main thread stack using the --main-stacksize= flag.
 11 ==12747==  The main thread stack size used in this run was 8388608.
```
4. 需要把优化开的比较低，编译器可能会优化掉从而让即便没有free也不会造成内存上的泄露。
5. 100显然是超出范围了。Out of scope.
6. This can run, but of no sense. Valgrind can detect this error.
7. This will be an invaild pointer.

Other problems we will skip... :)
## Mechanism: Address Translation
Virtualizing memory goals:
1. efficiency.
2.  control while providing the desired virtualization.

General Idea: hardware-based address translation. -> redirect application virtual memory references to their actual locations in memory.

From the program's perspective, its address space starts at address 0 and grows. But the OS won't necessarily put them into address space starting at 0.

### Simple idea: Base & Bound.
Need two hardware registers within each CPU, one is called the base register, and the other the bounds. This pair is going to allow us to place the address space anywhere we'd like in physical memory, and do sso while ensuring that the process can only access its own address space. (can't access others' space).

$Physical \_ address = Virtual \_address + base$

**Relocation:** Translate VA to PA. This happens at runtime, so even after the process has started running, this will also work well.

base and bound registers are hardware structures kept on the chip, **and one pair per CPU.** The hardware conduct translation is always called MMU.

To support Base & Bound ability, 5 traits should be added for hardwares:
1. Privileged mode.
2. Base/Bounds registers.
3. Ability to translate virtual addresses and check if within bounds.
4. Privileged instructions to update base/bounds.
5. Ability to raise exceptions.

What OS can do?
1. Memory management.
2. Base/Bounds management. -> restore/save.
3. Exception handling.

Bad parts: To inefficent. -> Internal fragmentation.

## Segmentation:
Basically using base & bound in a large memory space will cause great waste, given that the process demands the entire address space be resident in memory.

So that is Segmentation, a generalized Base & Bounds method. Instead of having just one base and bounds pair in our MMU, why not have a base and bounds pair per logical segment of the address space? This will mitigate internal fragmentation.

As an example:
```
-------------------- 0 kB
|        OS        |
-------------------- 16 kB
|        ^         |
--------------------  
|      stack       |
--------------------
|     not used     |
--------------------
|       code       |
--------------------
|       heap       |
-------------------
|        |         |
-------------------- 48 kB
|      not used    |
-------------------- 64 kB
```
in the MMU, we can therefore maintain a table.

|Segment|Base|Size|
|-|-|-|
|code|32k| 2k|
|heap|34k|3k|
|stack|28k|2k|

When the code access out-of-bound memory, it will cause segmentation fault. **AND THE TERM PERSISTS.**

### Which Segment are we referring to?
1. Common approach: chop up the address space into segments based on the top few bits of the virtual address. **Used in VAX/VMS system.**

Use top bits for example, as the segment selection part. Below is the mapping of VA2PA.
```C
/*
* 13   11              0 *
* ---------------------- *
* |   |                | *
* ---------------------- *
* seg     offset.        *
*/
// get top 2 bits of 14-bit VA
Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT
// now get offset
Offset = VirtualAddress & OFFSET_MASK
if (Offset >= Bounds[Segment])
     RaiseException(PROTECTION_FAULT)
else
     PhysAddr = Base[Segment] + Offset
     Register = AccessMemory(PhysAddr)
```
### Growing directions.
Hardware also needs to know which way the segment grows, one bit should be provided to tag them as Postive grow or Negative grow.
### Support for Sharing.
code sharing is common and still in use in systems today.

Protection bits should also be set, to inform the kernel whether a segment is readable/writable/executable or not.
### Fine-grained vs Coarse-grained Segmentation.
The e.g. we mentioned above is Coarse-grained Segmentation, but in fact early machine like **Multics** are more flexible and allowed for address spaces to consist of a large number of smaller segments, which is referred ato as fine-grained segmentation.

Such a Fine-grained segmentation requries further hardware support, with a segment table of some kind stored in memory.

Fine-grained Segmentation will utilize main memory more effectively.
### OS support
1. Segment registers must be saved and restored.
2. Segment grow or shrink.
3. Manage free space in physical memory.

### Bad parts:
Segmentation still causes great fraction (inner and external), so instead of finding an algorithm to utilize these fractions more effectively, a better way might be systematically change our segment mechanism into a more advanced one.

## Free-Space Management
For 'free' function, the library must be able to figure out how big a chunk of memory is when handed just a pointer to it. A basic idea might be store metadata in the heap, therefore we will be able to release the exact amout of space.

See following code:
```C
// chunks in use.
typedef struct {
     int size;
     int magic; // magic number like ELF header magic number to check the intergrity.
} header_t;

// free space, we use node to link them together.
typedef struct __node_t {
     int size;
     struct __node_t * next;
} node_t;
```
### Basic Strategies to allocate space.
1. Best fit.
2. Worst fit.
3. First fit. -> possibility to pollutes the beginning of the free list with small objects exists.
4. Next fit. -> avoid pollution.
### Other approaches:

#### segregated lists: 
if a particular application has one (or a few) popular-sized request that it makes, keep a separate list just to manage objects of that size.

For area of the right size, this will definitely benefit a lot.

**Slab allocator:**dynamically switch between segregated lists and general allocator.
#### Buddy allocation:
√.
#### Other ideas:
scaling, trees, glibc allocator. -> See Papers.

## Paging: Introduction
Segmentation: chops things up into variable-sized pieces. This solution has inherent diffculties -> different-size chunks, leading to fragmentated pieces.

chop up space into fixed-sized pieces might be a better idea. -> Paging.

**Advantages:**

1. flexibility: a fully-developed paging approach, the system will be able to support the abstraction of an address effectively. (No Need to make assumptions about the heap & stack growing directions)
2. simplicity management.

record mappings: -> **page table**

most page table structures we discuss are per-process structures.
### Baseline Idea:
Basic virtual address form and translation:
```
PPN: or PFN, physical page number or physical frame number.
VPN: virtual page number.

          ------------------
          | VPN  |  Offset |
          ------------------
             |        |
             |        |
-----------------     |
|    address    |     |
|   translation |     |
-----------------     |
        |             |
---------------------------
|     PPN       |  Offset |
---------------------------
```
### Page Table Store machnism.
PTE: page table entry.

we use PTE to hold the physical translation plus any other userful stuff.

For every process we should maintain a set of translation mapping reference table, apparently it will cost a lot if our data structures are not well designed.

Page table is just a data structure that is used to map virtual address to physical addresses, to be more exact, map VPN to PPN.

The OS indexes the linaer page table, which is an array, by the virtual page number, and looks up the page-table entry(PTE) at that index in order to find the desired physical frame number.

An x86 page table entry looks like this:
```
31                   12 11 9 8  7  6 5  4   3   2   1  0
---------------------------------------------------------
|          PFN         |\\\\|G|PAT|D|A|PCD|PWT|U/S|R/W|P|
---------------------------------------------------------
```
It might be strange we got no Valid bit here in intel x86 PTE, but in fact is common in hardware design: we often just provide the minimal set of features upon which the OS can build a full service.

If the hardware are designed with too many features, it will also limit the OS developers.
### Paging: Still too slow.
If we still use a linear array to store our PTEs, the fetching is still very costly.

1. Two times to access memory.
2. Too large PageTable. 
```C
// Extract the VPN from the virtual address
VPN = (VirtualAddres & VPN_MASK) >> SHIFT

// Form the address of the page-table entry (PTE)
PTEAddr = PageTableBaseRegister + (VPN * sizeof(PTE))

// Fetch the PTE
PTE = AccessMemory(PTEAddr)

// Check if process can access the page
if (PTE.Valid == False)
     RaiseException(SEGMENTATION_FAULT)
else if (CanAccess(PTE.ProtectBits) == False)
     RaiseException(PROTECTION_FAULT)
else
     // Access is OK: form physical address and fetch it
     offset = VirtualAddress & OFFSET_MASK
     PhysAddr = (PTE.PFN << SHIFT) | offset
     Register = AccessMemory(PhysAddr)
```
### Memory Access counting:
When we fetch the instructions, we also need access the memory.

## Paging: Faster Translation (TLBs)
How can we speed up address translation, and generally avoid the extra memory reference that paging seems to require?

Speed up -> always comes from Hardware.

TLB is part of the chip's memory-management unit, and is simply a hardware cache of popular virtual to physical address translation. We can simply conclude it as address-translation cache.

The algorithm is given below:
```C
// get the VPN of the VA.
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
// Use VPN as index to lookup the TLB.
(Success, TlbEntry) = TLB_Lookup(VPN)
// If we can find VPN in TLB, it will cause a TLB Hit.
if (Success == True)
     // Check TlbEntry's ProtectBits.
     if (CanAccess(TlbEntry.ProtectBits) == True)
          Offset = VirtualAddress & OFFSET_MASK
          // TlbEntry will have PFN for us to use.
          // Now we can assemble it.
          PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
          Register = AccessMemory(PhysAddr)
     else
          RaiseException(PROTECTION_FAULT)
else
     // fail to find TlbEntry.
     // So we simply use the startegy in last chapter.
     PTEAddr = PTBR + (VPN * sizeof(PTE))
     PTE = AccessMemory(PTEAddr)
     // In intel, Valid & Protect Bits are seen as the same one.
     if (PTE.Valid == False)
          RaiseException(SEGMENTATION_FAULT)
     else if (CanAccess(PTE.ProtectBits) == False)
          RaiseException(PROTECTION_FAULT)
     else
          // That's of course, think about the cache.
          TLB_Insert(VPN, PTE.PFN, PTE.ProtectionBits)
          RetryInstruction()
```
### Who Handles the TLB Miss?
It depends, in CISC machine like x86 architecture, which uses a fixed multi-levl page table, the hardware didn't much trust those sneaky OS people. In these cases, it is hardware that handles the TLB Misses.

More modern architectures like MIPS that use RISC ISAs will use trap handler to do this stuff. The TLB miss will be handled by the software.

The return-from-trap instruction is different than the return-from-trap we saw before when servicing a system call. Two things should be handled carefully.

1. After handling TLB miss, the hardware must resume execution at the instruction that caused the trap. This retry lets the instruction run again, and this time will result in a TLB hit. 

So the hardware must save a different PC when trapping into the OS to properly resume.

2. Besides, the OS must be careful not to cause an infinite chain of TLB missses to occur.
### TLB contents:
```
-------------------------
| VPN | PFN | other bits|
-------------------------
```
other bits will store interesting flags for writable/readable/executable and etc..
### TLB context switches issue
TLB contains virtual-to-physical translations that are only valid for the currently running process, these translations are not meaninful for other processes.

ways to solve this issue:
1. flush the TLB on context switches, thus emptying it before running the next process.
2. provide an address space identifier(ASID) field in the TLB to mitigate great cost in flushing.
```
-----------------------------------
| VPN | PFN | valid | prot | ASID |
-----------------------------------
| 10  | 100 |   1   |  rwx |  1   |
|  -  |  -  |   0   |   -  |  -   |
| 10  | 170 |   1   |  rwx |  2   |
|  -  |  -  |   0   |   -  |  -   |
-----------------------------------
```
### cache replacement:
when we are installing a new entry in the TLB, we have to replace an old one, and thus the question: which one to replace?

One Key idea: minimize the Miss Rate.

common approach: 
- least-recently-used -> LRU entry. 
- random policy.

MIPS R4000 with software-managed TLBs: 32-bit address space with 4KB pages
- that means 20-bit VPN and 12-bit offset in typical virtual address.
```
63                           45  44 43   40 39         32
---------------------------------------------------------
|          VPN                 | G |\\\\\\\|    ASID    |
---------------------------------------------------------

31 30 29                              6 5     3  2  1  0
---------------------------------------------------------
|\\\\|                 PFN             |   C   | D| V |\|
---------------------------------------------------------

VPN: 19 bit, user can only use half of the VPN.
G: global shared bit.
ASID: if G is set, this field will be ignored (of course)
C: Coherence bits. Which determines how a page is cached by the hardware.
D: dirty bit.
V: valid bit.
```
OS can also use part of the TLBs(critical part), the OS uses these reserved mappings for code and data that it wants to access during critical times, where a TLB miss would be problematic(Of course!).
### Sum.
TLB bottleneck -> VIPT cache.

See CSAPP.

## Paging: Smaller Tables
We've fixed problem of slow speed of page table, but page table is still to big and thus consume too much memory.

A simple solution might be using bigger pages, therefore we not only can have better locality, but also get smaller page table.

If pages are of 16 KB size, we should have 18-bit VPN plus a 14-bit offset. If the PTE is 4 bytes, for each page table, then the total size will be 1 MB. Whereas the 4 KB page will have the total page table size of $2 ^ 20 * 4 = 4 MB$. That's a big deal.

In fact many architectures now support multiple page sizes, but this might do harm to the compatibility.
### Hybrid Approach: paging and segments.
`Multics` using segments to reduce the cost of paging strategy.

Since many pfns are in fact unmapped, instead of having a single page table for the entire address space of the process, why not have one per logical segment? So we might thus have three page tables, one for the code, heap, and stack parts of the address space.

So the virtual address may be treated as the following structure:
```
31  30 29                               12 11            0
----------------------------------------------------------
| seg |                VPN               |      Offset   |
----------------------------------------------------------
seg: 01 for code, 10 for the heap, 11 for the stack.

SN = (VA & SEG_MASK) >> SN_SHIFT
VPN = (VA & VPN_MASK) >> VPN_SHIFT
AddressOfPTE = Base[SN] + (VPN * sizeof(PTE))
```
However, using hybrid method here might not always be a good idea, for segments to some extent assume a certain usage pattern of the address space.
### A better way: multi-level page tables
Basic idea:
1. chop up the page table into page-sized units.
2. if PTE is not valid, don't allocate that page of the page table at all. -> We use page directory to track this.
3. use `page directory` to track whether a page of the page table is valid.

The following is a good example.
```
Multi-level page table
                    pd       pt      offset
Virtual address: ----------------------------
                 |  2bit |  2bit |    4bit  |
                 ----------------------------
      ----------------
PDBR: |      200     | -----
      ----------------     |
                           |
 valid   PFN               |
------------------- <------|                          valid prot   PFN
| 1|     201      | ------------------------------> --------------------
-------------------                                 | 1|     rx |  12  |
| 0|              |                                 --------------------
-------------------                                 | 0|    rx  |  13  |
| 0|              |                                 --------------------  PFN 201              
-------------------                                 | 0|        |  -   |                                 
| 1|    204       |------------------               --------------------                                 
-------------------                 |               | 1|    rw  | 100  |                                 
                                    |               --------------------                  
                                    |
                                    |
                                    |------------> .......                             
```
每一级页表均可以用一个页来存储。

Multi-level bad parts: 
1. on a TLB miss, two loads from memory will be requried to get the right translation information from the page table. One for the page directory and one for the PTE itself. (Pages are stored in memory). But a linear one only cost one load.
2. Complexity.

If we want to access the PTE of a virtual address in multi-level page, the process might be like this:
```
     ------------------------------------------------------------
VA:  | Page Directory Index | Page Table Index |     offset     |
     ------------------------------------------------------------

1. Get the PDE addr: this is in fact a PTE.
PDIndex = high bits of VPN in virtual address.
PDEAddr = PageDirBase + (PDIndex * sizeof(PDE))

2. Access the PDE.
PDE = Access(PDEAddr)

3. Get the PPN of the next level.
PDE.PPN

4. Get PTEAddr. (PPN partial reflect a part of the address of PTEAddr, here it acts like the Base)
PTEAddr = (PDE.PPN << SHIFT) + (PTIndex * sizeof(PTE))
```

![](http://rcore-os.cn/rCore-Tutorial-Book-v3/_images/sv39-full.png)

### Inverted Page TABLES.
Very similar to RMP in AMD SEV-SNP. 看来也都是站在巨人的肩膀上。

## Beyond Physical Memory.
Why do we want to support a single large address space for a process?

> Once again, the answer is convenience and ease of use. With a large address space, you don't have to worry about if there is room enough in memory for your program's data structures.

Swapping space allows the system to pretend that memory is larger than it actually is. Process can store some parts of its memory into the disk. To satisfy this demand, we must add even more machinery: When the hardware looks in the PTE, it may find that the page is `not present` in physical memory. The way the hardware determines this is through a new piece of information in each page-table entry, known as the `present bit`.

If the PTE's present bit is 0, the page is not in memory but rather on disk somewhere. -> Page Fault Handler.
### Page fault.
This always is handled by OS for perfs and simplicity.

Process:
1. Get the PFN of the page for a disk address. -> find a free one.
2. Faults come, looks in the PTE to find the address, and issues the request to disk to fetch the page into memory.
3. Disk I/O completes.
4. update page table to mark the page as present, update the PFN field of the page-table entry to record the in-memory location of the newly-fetched page.
5. TLB miss & TLB updates.
6. finish.

What hardware does? TLB or Page access.

What OS does? Handle exceptions.
```C
PFN = FindFreePhysicalPage()
if (PFN == -1)
     PFN = EvictPage()
DiskRead(PTE.DiskAddr, PFN)
PTE.present = True
PTE.PFN = PFN
RetryInstruction()
```
### Replacement Occurs:
high watermark (HW) and low watermark (LW)

When the OS notices that there are fewer than LW pages available, a background thread that is responsible for freeing memory runs, the thread evicts pages until there are HW pages available.

This thread -> swap daemon or page daemon.

Cluster or Group a number of pages. (reduces seek and rotational overheads of a disk, increase performance noticeably).

When using `swap daemon`, the process will be changed a bit (use the thread to free the pages, and reawken the original thread beforehand).

