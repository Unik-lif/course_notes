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


