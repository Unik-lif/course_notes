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

