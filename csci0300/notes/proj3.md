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
int rwtail; // 用于记录cache中读写头终结的位置，以此判断是否充满了cache
int fprefetch; // 用于记录cache中预期数据尾端所在的位置，以此判断是否cache内先前读了现在所需的数
int fsofar; // 用于记录在用这一份cache之前以前所达到的文件偏移量
int foffset; // 用于记录文件中读写指针的位置
```

#### 基本策略与考虑
虽然针对read和readc的实现应该会有所不同，不过为了方便起见，这里打算把可能的情况都在readc之中进行讨论，之后构造read就只要调用readc的接口就好了。

唯一的代价是在readc环节需要做更多的细节考虑。

相应的，我们希望在writec中也有类似的方法，这样能够减小开发难度。
#### 小的问题：
本人在以前基本很少出现同时对于文件执行读写的操作，所以需要写个demo稍微了解一下同时对于文件执行读写的效果是怎么样的。
```C
#include <stdio.h>

int main() {
        char c;
        char d = 'b';
        FILE* fp = fopen("text", "r+");
        fread(&c, 1, 1, fp);
        printf("c = %c\n", c);
        fwrite(&d, 1, 1, fp);
        fread(&c, 1, 1, fp);
        printf("c = %c\n", c);
        fclose(fp);
}
```
其中test文件如下所示：
```
akkkdefghi
```
运行后得到结果如下所示：
```
abkkdefghi
```
因此说明了一个很关键的点。对于`"r+"`，如果不是一开始在最后位开始操作，即一进来就开始append，不论读写对原文件的长度，并不会产生改变。

但对于`w+`的效果则是truncate，类似于推倒重建。两种格式的表现很不一样，所以我们需要简单看一下项目希望我们达成哪个flag。

破案了，需要完成的是`"r+"`功能，也就是说我绕了个圈子哈哈。
#### 小小问题
完成了readc和writec以及lseek函数的撰写，在修改read和write函数之时，由于先前的路线认为完成好freadc就可以直接延展到read函数了，但实际上依旧过不了一些测试点，这说明其实还没有这么简单。

下面是一个很小的demo。
```C
#include <unistd.h>
#include <sys/stat.h>

int main() {
        int i = 99;
        char c = 'a';
        int filesize;
        FILE* fp = fopen("text", "r+");
        struct stat s;
        int const r = fstat(fileno(fp), &s);
        if (r >= 0 && S_ISREG(s.st_mode)) {
                filesize = s.st_size;
        }
        fseek(fp, filesize, SEEK_SET);
        i = fread(&c, 1, 1, fp);
        printf("c = %x i = %d\n", c, i);
        fclose(fp);
        return 0;
}
```
即便在我事先把文件偏移量搞到，将fp移动到最后一位的情况下，采用read再读一位，其实也不会出现错误，最终能够返回`61`这个值。而且，最终的`i`也不会返回一个异常值，而是直接返回0。

这个异常值其实才是最为关键的地方，我们后知后觉地在RTFM后看到了这么一句话：
```
On  files  that support seeking, the read operation commences at the file offset, and the file offset is incremented by the number of bytes read.
If the file offset is at or past the end of file, no bytes are read, and read() returns zero.
```
也就是说，除非出现异常错误，否则read不会返回-1这个值，而是返回0这个值。而与之相对的readc与writec则在完成读写操作之后很自然地返回-1这个值。

虽然利用readc和writec进行read和write函数的撰写操作是相对来说比较轻松和愉快的，但由于进行了两层嵌套，最后的性能表现并不是很好。

重写了一遍，性能提高了不少：

最终结果如下所示
```
======= PERFORMANCE RESULTS =======
byte_cat: 6.55x stdio's runtime
reverse_byte_cat: 4.73x stdio's runtime
block_cat: 2.0x stdio's runtime
reverse_block_cat: 1.5x stdio's runtime
random_block_cat: 3.0x stdio's runtime
stride_cat: 2.78x stdio's runtime
```