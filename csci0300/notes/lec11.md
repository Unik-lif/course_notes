## lec11:
stack frames on `x86-64` linux are aligned to 16-bytes boundaries.

一个最好的cache替换算法：`Bélády's optimal algorithm`

关于`prefetching`，这本质上就是很典型的可以被幽灵漏洞搞一搞的东西。

## lab4：
cache策略运用与否：
```C
int fd = open("data", O_WRONLY | O_CREAT | O_TRUNC | O_SYNC, 0666);
```
可以在文件描述符中添加`O_SYNC`这个负荷，该负荷会要求对于`fd`的访问需要在整个系统的全部存储等级中同步相关的修改信息后才可以继续执行任务。如果我们去掉`O_SYNC`标识符，则会让我们的工作对于脏数据也不会停留工作，这边的读写即为异步。

如果我们不使用`open`这一类需要叨扰操作系统的系统调用，而是使用`fopen`等库函数，访问的速度会显著地提升。原因是`fopen`,`fwrite`等函数其自身在库中存在自己的`cache`，能够很迅速地把结果返回给调用了他们的应用程序。


后续涉及cache的实验内容已经在cs61c中实现过，此处就不再赘述。
## Lec 13:
On UNIX-like operating systems such as macOS and Linux, there are some standard file descriptor numbers. FD 0 normally refers to stdin (input from the terminal), 1 refers to stdout (output to the terminal), and 2 refers to stderr (output to the terminal, for errors). You can close these standard FDs; if you then open other files, they will reuse FD numbers 0 through 2, but your program will no longer be able to interact with the terminal.

在Unix系统中的文件描述符0，1，2，均有妙用，而再去打开一个文件，则我的fd将从3开始向下计数。

## Lec 16:
Page Table策略不仅仅是为了效率，也是为了安全，其成为主流的方案确实存在很多优越的地方。

在WeensyOS中，进程通过ptable进行管理，每一个进程均会链接着一个多级页表链。

![](https://cs.brown.edu/courses/csci0300/2022/notes/assets/l14-page-tables.png)

### 虚拟地址转换：
Virtual Address Translation
The x86-64 architecture uses four levels of page table. This is reflected in the structure of a virtual address:
```
63         47     38     29     20     11        0
+---------+------+------+------+------+-----------+
| UNUSED  |  L4  |  L3  |  L2  |  L1  | offset    |
+---------+------+------+------+------+-----------+
          |9 bits|9 bits|9 bits|9 bits| 12 bits
```
每一级页表的表项存放着下一级页表的基地址，以此向下逐步索引。最后一级页表则是直接fetch到我的

## Lab 5：
其实是一个很简单的实验，多折腾折腾就完事了。可以利用这个帮助我们理解WeensyOS中的有意思的事情。
## Lab 6：
多进程管道实验，有一些些小收获：比如再次在'\0'上裁了跟头，乐死我了。

最终的实验结果大致如下所示：
```
开了多进程的话最终计时结果如下：
------------------------
Enter search term: 
real    0m13.753s
user    0m32.804s
sys     0m3.218s

顺序执行的话最终计时结果如下：
------------------------
Enter search term: 
real    0m26.592s
user    0m22.088s
sys     0m0.669s
```
实际经过的时间得到了缩短，说明确实多进程加快了程序的执行速度。
## Lec 18:
concurrency与parallelism的区别：

前者其实是time-sharing的分时利用，而后者不一样，它表示的是同时。



| Resource	| Processes (parent/child)	| Threads (within same process) |
| ----- | ----------- | -------|
| Code	| shared (read-only)  |	shared |
| Global variables	| copied to separate physical memory in child	| shared|
| Heap	| copied to separate physical memory in child |	shared |
| Stack	| copied to separate physical memory in child |	not shared, each thread has separate stack |
| File descriptors, etc.	| shared	| shared |

总而言之，对于进程而言，除了代码和file descriptor，其余都有点不同。而线程则仅有自己的栈是不同的。

C++的动态内存，即HEAP恰好是线程们均能看到的位置。因此我们更加推荐采用堆动态分配空间以在多个线程之间进行共享。
## Lab7：
在C++中利用vector来存放进程时，需要使用emplace_back比较合适。

线程在C++和Rust中似乎都不允许进行拷贝操作，为此哪怕是利用emplace_back来设置，也应该是这个样子来创立：
```C++
vec.emplace_back(thread(func, args1, ...))
```
在Rust中似乎没这个麻烦，我直接进行clone拷贝一份就好了，非常轻松。

此外，在Lab7有一个非常有意思的细节：

我们用wordindex来做的时候，需要注意其应该分配在堆上。如果它不是分配在堆上的，则会有一些毛病。比如，没法被其他线程所知道。虽然在我们这边的例子中这或许不会带来太大的问题，但是address sanizer说这样可能会造成一些内存泄露的问题。

此外，有一个bug我们de不出来，不过我目前貌似是明白了为何如此：
```c++
  for (int i = 0; i < num_threads; i++) {
    wordindex* index = new wordindex;
    index->filename = filenames[start_pos + i];
    //printf("filename: %s, word: %s\n", index->filename.c_str(), word.c_str());
    vecofThreads.push_back(thread(find_word, index, word));
    printf("index: word: %s count:%d\n", word.c_str(), index->count);
    total_count += index->count;
    files.push_back(*index);
  }
```
我们过早地将index塞到了files中，而此时很有可能子线程对于index的更新并没有完成，从而造成主线程读出来的东西存在滞后性。

特别感谢chatgpt的鼎力帮助。

做完这个实验后，原本关于线程执行前后这样的问题的方法论变得清晰了起来，我们来看看最后的实验结果吧。

```
多线程来跑
-------
Enter search term: 
real    0m12.366s
user    0m29.724s
sys     0m1.561s

顺序执行
-------
Enter search term: 
real    0m27.770s
user    0m23.818s
sys     0m0.713s
```

玩得够久了，没想到玩玩这个东西花了三小时，牛魔，今天剩下时间得all in科研了，不然对不起我的老师。

## Lab 8：
这个Lab很简单，我这边没有记录，就这样吧

## Lab 9:
值得记忆`Protobufs`和`gRPC`这两个概念。

What makes Protobufs convenient is that you only write a high-level description of the data you want to encode, and then the Protobuf compiler generates much of the encode/decode logic for you, in the language(s) of your choice. So for example, you could have a C++ server talk to clients written in Java, Go, or Python. As long as both the server and client are using the generated functions to encode/decode the Protobuf messages, they can understand each other!

`gRPC`常常用`Protobufs`来表示数据

目测这个实验和泛型之间的关系非常紧密，而我第一次看到泛型还是`Java`和`Rust`，不得不说如`Rust`语言它天生就有一些作为新事物的优越性质。