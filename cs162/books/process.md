## 阅读记录
### Chapter 2
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

系统调用的时候，存在一些边界问题，有可能会被TOCTOU攻击，为此，在内核中会有一个叫做kernel stub的存在，用于保证系统调用时的可用性
- 检查userStackPointer中所示的地址是否正确
- 做完检查之后，拷贝到内核本地地址，并比较相关的内容是否被篡改了，需要确保正确

Upcalls: 由Kernel发送给用户的一些事情
- 由某些事件触发user程序本身的注意力
- 线程之间的平均共享，利用timer来做
- 由user process自行来处理exception事件
- signals: user地址自动存放上下文信息

user signal处理流程
- 由kernel把保存的状态的信息拷贝到signal stack的底部
- 重新设置保存状态使得其指向用户的signal stack部分
- 利用iret跳转到用户层面，由用户的signal handler来处理，处理放在用户的signal stack中
- signal handler返回的时候，处理器状态将先重新被拷贝到kernel memory中，再利用iret返回到用户进程里

由于BIOS程序会写到ROM中，不能被修改，而OS的更新往往是很多的。为此，工程师们只是会把小部分的代码放到BIOS中
- 上电之后，从ROM读取BIOS代码
- BIOS从磁盘中拷贝bootloader
- bootloader拷贝操作系统内核
- 操作系统内核把login app拷贝到内存中

为了通过完整性检查，BIOS还会用密码学签名来防护之

虚拟机这边：本质上是模拟真实硬件中，跑guest operating system的效果
- host kernel一定程度上扮演起了硬件角色，会负责原本硬件做的存储上下文到特定位置的工作
- 由用户进程产生的exceptions，host kernel会直接移交给guest kernel处理
- 但是由guest kernel触发的exceptions，host kernel会自己来做模拟
- 因此host kernel必须要track virtual machine当前运行的状态
- timer时钟由host kernel自硬件捕获，然后模拟硬件，跳转到guest kernel的时间interrupt处理函数中

IO方面，则直接让host kernel拷贝文件信息到对应的guest kernel memory中，效果就像真的DMA hardware一样

特别的，新的处理器中，guest operating systems甚至可以自己直接来处理异常请求

### Chapter 3
需要搞清楚哪些东西是放到OS中，哪些东西是放在User-Level的library中

Thin waist模型：保持OS library和开发中间件的大体一致，从而促使应用和硬件的解耦，以及更进一步的发展

User与OS的边界主要考虑下面几个因素
- 灵活度, syscall语义要更加固定，但能表示很多
- 安全性，资源管理和防护需要交给OS
- 可靠性，轻量化的内核可靠性一般会更好
- 性能，如果总是去kernel，会有较大的代价

设计system call interface其实并不容易，在不同操作系统中的各个不同环节，在哪个部位体现OS的参与度和功能性是一个很重要的问题
