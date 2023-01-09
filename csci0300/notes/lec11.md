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
