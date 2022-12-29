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



## 一些小发现：
`struct`关键字在c++中似乎是不必要的，这挺不错的。