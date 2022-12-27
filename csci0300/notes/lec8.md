## Lec8:
### 优化与unsigned ints
优化后对于循环的判断条件会出现改变。

以程序`ubexplore2.c`为例子，该程序会将判断结构先对i进行加1，后执行框内的代码。这是在缺少一次判断下的默认优化。在极端情况下，可能会出现判断数的overflow现象。

https://cs.brown.edu/courses/csci0300/2022/notes/l08.html
## Lec9:
数值上的计算会影响`%rflags`寄存器的一部分值，常见的寄存器如下所示：
- ZF (zero flag): set iff the result was zero.
- SF (sign flag): set iff the result, when considered as a signed integer, was negative, i.e., iff most significant bit (the sign bit) of the result was one.
- CF (carry flag): set iff the result overflowed when considered an unsigned value (i.e., the result was greater than 2W-1 for a value of width W bytes).
- OF (overflow flag): set iff the result overflowed when considered a signed value (i.e., the result was greater than 2W-1-1 or less than –2W-1 for a value of width W bytes).

特别注意到`CF`和`OF`，他们分别对待unsigned类型和signed类型的变量。

对于内存上的全局变量访问行为，x86指令集中提供了一种`position-independent code`类型。该类型能够帮助程序在地址不固定的情况下依旧正确访问相关的全局变量。
```C
symbol_name(%rip)	%rip-relative addressing for global (see below)
symbol_name+4(%rip)	Simple computations on symbols are allowed (the compiler resolves the computation when creating the executable)
```
其中的`symobl_name`就是全局变量的名称。

%rsp：stack pointer，位于栈顶，也就是最低的地址位置处。

%rbp: 位于栈底，在高地址处。