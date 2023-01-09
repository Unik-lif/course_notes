## 知识回顾
static关键字往往用于将数据存在`data`段，而这个段一般是用来给全局标量和含有`static`的变量存放的。如果全局变量没有被初始化，则该部分变量会存放在`bss`段。

### Questions:
1. Describe a situation when it is much better to use dynamic memory allocation over static or automatic memory allocation.

当我们需要对某些较大的变量做长期操作的时候，利用`static`会占用较多的`data`段空间。此外，利用`static`关键字对于变量的操作和使用来说不会太灵活。

2. How does the heap grow in memory? How is this different from how the stack grows? Why is this the case?

堆一般是从低地址向高地址增长。这么做主要是为了内存空间使用的方便，如果和`stack`一样从高地址向低地址增长，对于内存的分配会变得有限，可能要设定固定的边界，而面对`stack`和`heap`其中有一个较大的情况则不会灵活。总而言之，堆与栈相反的增长方式让内存的使用效率变得更好。

3. Consider some pointer, ptr, that points to memory address 0x12f99a64. What address does (char *) ptr + 1 point to? What address does (int *) ptr + 1 point to? Assume 32 bit (4 byte) integers.

前者会指向`0x12f99a65`，而后者会指向`0x12f99a68`。

4. Name and describe two potential memory errors that can arise from using malloc and free incorrectly.

其一是`double free`，从而造成内存信息的泄露。其二是没有正确分配`malloc`大小，从而造成`segment fault`。

5. How is checking for integer overflow in addition different than checking for integer overflow in multiplication?

对于`addition`，注意最高一位`bit`的转变，比如两个正数相加变成了负数，两个负数相加变成了正数。

对于`multiplication`，检查会稍微复杂一些：我们可以将数字分成两部分，分别进行乘法操作验证。

### 关于`malloc`和`free`函数的细节：
`malloc`函数可以这样用：`malloc(0)`，与其余`malloc`和栈内元素不重叠的空间，并且返回一个正常的指针地址值，也可以通过`free`进行释放。

## 一些坑点与尝试：
在`phase 1`中，我们做了一个很有技巧的做`malloc`工作的监督尝试：多分配一点空间，然后把分配了多少空间作为变量写到堆中，便于追溯内存堆的工作。

此外，在`phase 2`中，面对溢出的情况，我们可以采用在`Question`部分提及的查询方式帮助我们较好处理`malloc`和`calloc`分配较大空间时可能造成的问题。

接下来是`phase 3`，通过检查测试集我们发现`heap_min`和`heap_max`如课程网站注释所言，均是**曾经**访问过的边界值，因此不能帮助我们追踪堆的调用链。我们希望通过链表完成这件事情，而有”哨兵“的链表用起来会很舒服，我又不想把它放在堆上（这样就破坏了美感），因此我决定在这里加上一个全局变量来帮助我定位。

在这一步中添加了`double free`,`not allocated`等相关错误检查的机制。

之后，是`phase 4`，采用的想法类似于缓冲区溢出的`canary`金丝雀操作。我们尝试做好这一件事情。

后续花了若干时间后：

值得注意的是，在第33号测试点花费了我一点时间。这个测试会将metadata清除掉，为了避免对拍错造成影响，我在栈上专门搞了个东东，这使得我能够通过除了34测试点以外的全部测试点。

但是出现了一处性能瓶颈，而且一些简单的优化并没有取得较好的效果，这使得我考虑重构原来的代码（实际上就是重写一遍吧哈哈）。为了把运行时间从19s降到1s以内，这次将会利用unordered_map这一数据结构来做，该数据结构将会存放被malloc过后的一些信息，比如对于free掉的block设置成free状态，这样就能在检测double free时发挥较好的效果了。

这样一来，设计时我们的数据结构唯一出现的遍历就是定位指针时的工作了。

metadata是要保留的，不然真的会很麻烦（哭了，果然阳性脑袋没有什么才智）。
## 1月2日的修改；
在1月2日乘着复习无聊的劲儿重构了一下先前的代码，利用unordered_map的哈希查找让速度变得很快，足够通过test034了。

但是test033再次成为了问题：注意到在malloc向unordered_map插入元素时的堆值很受到影响，很可能会在赋值的附近。

为了避免这个问题，我们需要把原来的元素给**删去！**，然后再重新插入，这样就不会在原来的空间附近了！

完美利落地解决了！
## 一些小发现：
`struct`关键字在c++中似乎是不必要的，这挺不错的。

C++标准库元素存储的地址和对应的堆内排布方式。（当然，这个是经验）