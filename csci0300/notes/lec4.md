## Lec4 & Lec5 & Lec6 & Lec7
### Lec4: lifetime:
lifetime将会决定变量存储的region位置。

全局变量在全局可见，其拥有static lifetime。

局部变量仅在函数内可见，其拥有的是automatic lifetime。

还有第三者：dynamic lifetime。

注意相关的内存排布与变量排布方式，特别的，dynamic lifetime variable在上述二者区域之间，因此我们可以画出内存中数据的排布大致图像。

### Lec5: struct结构体的初始化:

`struct x_t my_x = { 1, 2, 3, 'A', 'B', 'C'};`, or even only partially initialize the struct via `struct x_t my_x2 = { .i2 = 42, .c3 = 'X' }`;
### Lec6: sizeof相关
sizeof函数仅是在编译时使用的，这也意味着动态分配的空间大小是没法通过这个函数得到的。

很经典的内存泄露问题会由此引发，为此，要小心不要让他去读某个指针的大小，无论如何返回的结果估计都是8或者4。


| Type	| Size	| Address restriction| 
| ---- | ----	| ---- |
| char (signed char, unsigned char)| 1	| No restriction |
| short (unsigned short) | 2 | Multiple of 2 |
| int (unsigned int) | 4 | Multiple of 4|
| long (unsigned long) | 8 | Multiple of 8 |
|float	|4	|Multiple of 4|
|double	|8	|Multiple of 8|
|T*	|8	|Multiple of 8|
### Lec6: 变量内存排布相关
变量之所以会出现对齐现象，实际上是为了cache方便抓取数据。试想如果一个数据横跨了两个cache block，那么底层电路会因此变得非常复杂，因为要涉及数字的拼装，运行速度也会很慢。

为此，定义了上述的address restriction规范。

在关闭优化时，我们将会看到编译器的reorder能力失效。否则在栈中声明一组变量时，会被编译器reorder，让其内存的利用率更高。

但是，在采用struct结构体进行包装的时候，struct严格要求内存布局如其声明所示，这里的内存排布不会被编译器优化。、

### Lec7: Malloc rules
任何malloc分配的指针在十六的倍数处才开始分配，也就是说：

any pointer returned from malloc points to an address that is a multiple of 16.

### Lab3:
计算机读指令是从低地址读到高地址（

