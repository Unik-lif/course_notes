## 学习PPT
我们看向`Address Space`，这边有个幻灯片演示，里头有个`gdb`跟踪内核的流程，过程非常的`hacky`。

具体来说其观察了在内核态和用户态时的`cs`信息。还在内核态的时候，可以通过`GDT`寄存器，获得`GDT`表的位置。

`GDT`在`64`位寄存器中，似乎其`base`值不再是一个特别重要的值，去访问代码段信息的话，我们需要查阅`GDT`中的表项，这个表项需要通过`cs`信息获得。

`cs`是一个选择子加上`RPL`的结构，如下图所示：
```
0-1 bit: RPL, for CS only indicates the current priviledge.
2 bit: TI, select the GDT or LDT.
3-15 bit: indexes the segment descriptor table.
```
以`8`字节数组方式对`GDT`表做解析，我们可以获得对应`cs`指向的具体代码段信息。

切换到用户空间之后，似乎`GDT`所对应的位置与映射方式没有发生改变？这是一个值得记录的信息。
### Segments:
在现在的`Linux`中，段已经不会用来去定义栈、数据、代码的方位了，但是还是需要使用`TSS`段来定义用来处理系统调用的内核栈空间。

### 一些mappings模式
高地址内存可以建立较为随机的映射，方式繁多，可以操作线性内存写死映射，但似乎只能在`boot`的时候使用。可以建立`temporary`映射，也可以建立长期可以使用的映射。