## Conceptual Questions
1. Come up with a real world analogy to explain caching to someone who does not know what a computer is.

借用Dan Garcia的叙述：我人住在圣芭芭拉，想要吃德州的蔬菜，在加州首府萨克拉门托有一个杂货店能够满足我的进货需求。这样，我想搞来蔬菜吃，就可以直接取萨克拉门托，而不必去德州。

2. What are the benefits of having a standard file I/O API provided by the operating system? How does this help programmers who might not know what hardware their programs will be running on?

对于高级语言开发者，他们不需要太了解底层的一些细节。设置文档完整、功能清楚的API会把工作大大简化。

3. Read about the catchphrase “Everything is a file” that is central to the design of UNIX-like operating systems (e.g., Linux, macOS, Android). Why might this complicate caching I/O?

因为全部的东西都会被抽象成文件，而各种各样的设备存在较大的差异，却要用同一的cache联系在一起。因此，驱动相关的I/O cache要考虑很多硬件参数细节，才能让执行效果相对更好一些。

### Two Impl:
#### Navie:
采用最直接的系统调用来做，具体来说就是直接利用`open, lseek, read, write`等代价很大的函数，此外该实现并没有利用cache进行加速。
#### stdio.h：
利用类似stdio.h库中的函数来实现，具体来说使用了`fopen, fread, fwrite`等标准库函数，实现了利用cache进行加速的功能。
#### Difference:
```shell
cs300-user@1eaa89c2b89c:~/cs300-s22-projects/fileio$ time ./byte_cat /tmp/testdata /tmp/testout

real    0m8.338s // the total time spent.
user    0m1.103s // time spent in user space
sys     0m6.818s // time spent in the kernel
cs300-user@1eaa89c2b89c:~/cs300-s22-projects/fileio$ time ./byte_cat /tmp/testdata /tmp/testout

real    0m0.120s
user    0m0.107s
sys     0m0.011s
```
确实存在着天壤之别。
### Tasks:
#### 1. Understand io300.h
唯一值得注意的细节：
对于close，需要把RAM内的脏数据写回disk，因此在有cache的时候需要记得flush清除干净。
#### 2. Why the difference of the speed exists?
利用strace跟踪一下相关实现中read和write，我们可以观察到两个现象：
1. stdio.h库中对于write和read的调用显著少于naive，可以视作是在标准库中提供了一个buff，之后利用批处理减少了write和read的次数。
2. naive则有大量的对单字节的read和write操作，大大减缓了实际上的运行速度。

### 方案：
想方案真的花了我漫长的时间，对照简单看了一下cs61的材料，可能要有一个多小时吧，才大概把这个问题想明白。

我们需要在cache之中设置一个读头和写头，最后以读写尾作为终结。
```C
int rhead; // 用于告知程序员读的位置从何开始
int whead; // 用于告知写的位置从何开始
int rwtail; // 用于记录cache中读写头终结的位置，以此判断是否充满了cache
int foffset; // 用于记录文件中读写指针的位置
```