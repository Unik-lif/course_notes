## 应用视角的操作系统
`helloworld`程序其实一点都不小，在我们不使用动态库加载，利用`gcc --static`静态加载时，可以用`file`来看一下文件有多大，用`wc -l`来看一下一共有多少行。

利用`gcc --verbose`来进行编译信息的查阅。
```
    预编译      编译      汇编       链接
*.c   ->   *.i  ->  *.s   ->   *.o  ->  a.out
```

`starti`，这里提到运行`gdb`装载好的程序的第一条指令的方法。

`layout asm/src`.

真正最小的可执行程序，需要有系统调用辅助。 
## 汉诺塔问题
如何非递归地写出汉诺塔问题代码？

chatgpt居然能写对！

思想：把语言本身就当做是一个状态机，手动模拟函数栈帧

g，f互相调用的递归似乎也可以这么做
## 编译器
优化与Barrier，存在Barrier的地方似乎无法被穿过

BusyBox.

蒋老师实在是太酷啦！