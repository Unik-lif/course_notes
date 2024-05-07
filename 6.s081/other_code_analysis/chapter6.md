## Chapter 6
这边有句话说的很好。

- On the RISC-V this instruction is amoswap r, a. amoswap reads the value at the memory address a, writes the contents of register r to that address, and puts the value it read into r.

amoswap 读取位于 a 处地址的值，把 r 寄存器的值写到 a 处地址，并且把 a 地址处原来的值写到 r 之中。

- all code paths must acquire locks in the same order. The need for a global lock acquisition order means that locks are effectively part of each function's specification: callers must invoke functions in a way that causes locks to be acquired in the agreed-on order.

当锁获得的顺序需要在全局范围内考虑时，所有的函数都要在设计中考虑到这一点，其对于锁的获取一定是要按照既定的、协定好的顺序。

这章写的非常精彩，我好喜欢。