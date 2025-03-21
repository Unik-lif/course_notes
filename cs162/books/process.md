## 阅读记录
用词的变化：
- 一开始机器没有什么多线程的，所以就只有进程
- 后来出现了多线程的进程，所以出现了lightweight thread这种说法
- 最后达成了一致协议，出现了process和thread的明显区分

dual-mode的出现
- 一开始是用一个软件层的interpreter，对指令做逐步的解析，同时检查其安全性
- 后来发现大部分指令可能是无害的，不会对安全性造成影响
- 于是把interpreter的功能写到了硬件上，这就是user-mode和kernel-mode的由来

直接使用物理地址，并且采用Base & Bound方案的劣势
- heap和stack的大小在变化
- 内存没法共享
- 物理地址得写死，否则需要复杂的动态管理
- 内存碎片化

为了更好使用内存，添加虚拟地址这一层抽象

Interrupt一般都是异步的
- 当interrupt发生的时候，只需要有一个CPU core能够handle就行，其他的核继续做自己应该做的事情就可以
- interrupt的来源包括用户触发，I/O完成后触发，以及处理器内部触发
- 处理器间的异常用于多个核之间的同步与合作

Buffer descriptors和高性能I/O
- 存一个环形队列，作为buffer存储I/O请求，利用descriptor来给DMA来直接访问
- 这样就能存储请求，在处理I/O时，I/O还能继续运行

Processor exceptions
- 出现一些奇妙的行为，比如除零操作，GDB输入INT3操作等
- 某些操作需要让OS发送请求把其他核一并中断
- exception触发时的某些操作可以用于模拟较为古早的设备和硬件，这在虚拟机中很有用

很多操作系统似乎会给用户进程提供User-level的upcall能力，可以让用户进程接收到异步event的通知信息。

一些有趣的切换的设计考量
- 有限的entry points
- 原子性的处理器状态切换，mode，PC，stack，memory protection的切换需要原子
- 控制流需要能够restartable

异常发生时，需要进入Interrupt Vector Table中的handler函数
- 每个进程都需要有两个栈，一个用户栈一个内核栈
- 在user mode跑的时候，kernel stack时空的
- 准备跑，等待调度时，user cpu state会存放在kernel stack中
- 如果在等待IO，kernel stack中会存放更多东西，包括syscall handler和IO driver程序，自然，User state也会存放在kernel stack中等待被restore

早期因为内存比较贵，Unix会选择开少一点kernel stack

有一些异常处理程序很长，而异常处理的时候，我们可能要屏蔽异常，在处理完之后，再重新enable。可是，存放interrupt的buffer也是有限的，为了解决这个问题，interrupt现在一般被设置成两个部分
- 底部用于快速存储状态，然后继续接受新的请求
- 把请求送到顶部之后，底部就能重新enable interrupt了，而顶部会处理紧急的任务
- 在顶部处理时，底部可以处理一些更一般的任务，不那么紧急的任务

示例场景：网卡收包
​上半部：网卡触发中断，内核立即将数据包从硬件缓存拷贝至内存（sk_buff），并清除中断标志。
​下半部：通过 Softirq（NET_RX_SOFTIRQ）将数据包递交给协议栈（如 IP/TCP 层），最终由应用程序处理。

硬件快速处理
- SPARC准备一套register windows，跑中断时直接切，根本不需要存
- 进程切换时代价非常大（可想而知）
- Motorola 88000会把流水线状态全记录下来，但是这样其实也不大好，开销比较大
- Intel则是丢掉interrupt发生时后头的指令，处理完异常之后，重新执行一遍

中断处理这一部分，由于我自己确实自己实现过，就掠过了。