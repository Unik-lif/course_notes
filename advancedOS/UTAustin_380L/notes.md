## Lec1:
OS的研究本质上是对大复杂系统模块化的研究。

### Take home

#### How (and How Not) to Write a Good Systems Paper

##### 批评和考虑
对于Idea本身？
- 是否足够新？你又是怎么知道足够新的？（当前的问题背景）
- 你是否可以把新idea表达明确？如果你不能写一个段落出来，你自己对于Idea的理解到位了吗？
- 你希望解决的问题是什么？
- 你的Idea重量级足够吗？如果只是很小的问题，它可以很快写完，不值得发在SOSP上。
- 你当前的工作和现有相关工作显著不同吗？一个很显然的方法是不足够的。
- 你是否真的引用并且阅读了现有相关工作了吗？引用是否丰富并且合适？
- 对于前序工作的比较要详细和具体/
- 工作是否对于前序未经证实的观点做了重要的扩展和验证？

对于现实性
- 在一开始就声称系统是否真实存在。
- 系统的用途，以及用途是否能够支持Idea的重要性？
- 你的系统最好是真的，除非非常超前。

对于教训
- 你从自己的工作中得知了什么？
- 请把自己的经验教训说出来，你希望读者学到什么，以供他人学习，避免重蹈覆辙。
- 这些经验的适用面是怎么样的？在做总结和一般化的时候，请重新提出你的假设，当然要避免假设过于武断，假设要贴合现实。

关于选择
- 告诉你选择这一条方式的原因。揭示为什么你仔细选择后选择了这一条路径。
- 你的选择是否正确？如果不是，告知原因以及自己学到了什么，讲讲经验之谈。

关于上下文
- 你的假设是什么？是否现实合理？
- 你的工作对于假设是否有较为敏感的依赖？如果建立于不确定性上，工作将会贬值不少。
- 如果要提出模型，下定义是不够的，定理是需要的。

关于目标聚焦
- 讲背景知识要有所侧重
- 为读者提供足够多的背景知识

关于表述
- Idea需要清楚，且术语需要提前得到良好的定义
- 不能总是说xxxx我们将在xxxx环节揭示，能揭示就尽量揭示，别吊别人胃口，对读者记忆力也会造成负担
- 是否考虑其他组织文章的方式？可能需要想好文章的侧重点
- 摘要可以先写，作为文章侧重点的表达，写完文章后再来对比修改摘要
- 论文的完整性，不能容忍重要部分的缺失

关于写作风格：
- 你不好好写，我为什么要号好看？

Summary

> These thirty-odd questions can help you write a better technical paper. Consult them often as you organize your presentation, write your first draft, and refine your manuscript into its final form. Some of these questions address specific problems in "systems" papers; others apply to technical papers in general. Writing a good paper is hard work, but you will be rewarded by a broader distribution and greater understanding of your ideas within the community of journal and proceedings readers.

#### On Being the Right Size (1926)
这是一个很好的观点，讲了变大或者变小各自的优势，当然也强调了成为恰如其分大小的难得。

#### You and Your Research
Not all luck, courage, curosity

有时候研究条件差可能反而会使你更容易去创造

要合适地努力，有选择方向的，不能盲目努力

找important problem的关键不是这个问题特别难，而是这个东西是reasonable attack

> By important I mean guaranteed a Nobel Prize and any sum of money you want to mention. We didn't work on (1) time travel, (2) teleportation, and (3) antigravity. They are not important problems because we do not have an attack. It's not the consequence that makes a problem important, it is that you have a reasonable attack.

> Instead of attacking isolated problems, I made the resolution that I would never again solve an isolated problem except as characteristic of a class.

> Most of the time the audience wants a broad general talk and wants much more survey and background than the speaker is willing to give. As a result, many talks are ineffective. The speaker names a topic and suddenly plunges into the details he's solved. Few people in the audience may follow. You should paint a general picture to say why it's important, and then slowly give a sketch of what was done. Then a larger number of people will say, ``Yes, Joe has done that,'' or ``Mary has done that; I really see where it is; yes, Mary really gave a good talk; I understand what Mary has done.'' The tendency is to give a highly restricted, safe talk; this is usually ineffective. Furthermore, many talks are filled with far too much information. So I say this idea of selling is obvious.

> Friday afternoons for years - great thoughts only - means that I committed 10% of my time trying to understand the bigger problems in the field, i.e. what was and what was not important. I found in the early days I had believed `this' and yet had spent all week marching in `that' direction. It was kind of foolish. If I really believe the action is over there, why do I march in this direction? I either had to change my goal or change what I did. So I changed something I did and I marched in the direction I thought was important. It's that easy.

给自己思考的时间，不要一味往前。关键是搞清楚重要的问题是什么，防止自己走错方向。

> If you really want to be a first-class scientist you need to know yourself, your weaknesses, your strengths, and your bad faults, like my egotism. How can you convert a fault to an asset? How can you convert a situation where you haven't got enough manpower to move into a direction when that's exactly what you need to do? I say again that I have seen, as I studied the history, the successful scientist changed the viewpoint and what was a defect became an asset.

认识自己的长处，有时候利用自己的短处去学习

## Lec2:
### In Class
#### The UNIX Timesharing System
主要介绍了Unix的长处，以及一些特色的介绍，不过我们本身对Linux很熟悉，这些东西技术点就别看了

来聊聊为什么这个东西很成功
- Designed the system to make it easy to write, test, and run programs => Interactive
- Hamming所说的，某些限制下的工作，salvation through suffering
- 及时修改，所有的应用程序都跑在shell以外

看起来也实现了Filesystem和内存的解耦，关键问题是并没有把memory的解读定死，而是使用untyped方式来做，提高了普适性
#### END-TO-END ARGUMENTS IN SYSTEM DESIGN
底层机器就应该做好底层机器应该做的事情，不应该尝试添加一些更高级的功能，否则性价比会变得非常低

在交互系统中，函数与程序的一些具体功能需要在端侧才能较好地实现，因为它们更清楚本身的需求。但是，把这一类的feature放置到communication system本身并不是一个良好的设计，但如果system中能够部分支持相关的能力，这将会提高performance enhancement

这个判断我只能说确实一针见血

但其他的具体的例子就没必要看了，太老了
### Take Home
#### Linux Edge
系统梳理了Linux的一些挑战和开发的体验

其中也许是最宝贵的经验：
- 屎山的管理
- 接口的设计
- 多架构的支持
- 但还是认为微内核是一坨屎

期望：避免一些一开始Linux撰写时的历史包袱
#### THE
这篇文章是迪杰斯特拉写的，核心是在论文中讲了设计多进程操作系统的一些思考。

核心思考是其通过分层测试和架构测试，来大大降低了工作的难度，通过模块化的处理，完成了这项较为困难的工作。

此外，还在论文中提出了关于信号量的概念，具体细节已经在OSTEP中阅读过，此处就不再赘述了。

#### On Micro-Kernel Construction
主要是指出，微内核架构并非性能很慢的直接原因，有很多是实现上的事情

当时的研究认为造成微内核性能很差的原因主要是地址空间和kernel-user状态的切换所带来的开销

文章先提出了一部分合理的微内核设计所需要达到的设计目标
- 独立性：各个模块之间可以不相干扰，可以各自作为机器的子系统
- 完整性：各个模块进行交互的信道不会受到其他模块的篡改

每一个子系统都拥有其所对应的专门负责的地址空间，并有下面的操作权限，因为涉及一部分所有权的转移，其实很像Rust生命周期管理的那一套
- Grant：分配页给其他子系统
- Map：映射名下的页给其他子系统
- Flush：清除名下的页

就thread和IPC层次来说
- thread要有自己的地址空间
- threads之间通过IPC进行交互
- 上面的内存操作也需要依赖IPC
- Interrupts其实也需要IPC的交互

关于切换的代价
- kernel和user的切换实际上是比较快的，需要谨慎测量，而且也要有较好的实现
- address space的切换也可以得到加速，先前对于这个切换过程觉得比较慢，很多情况是实现的问题
- IPC和Thread Switches也不见的很慢，一些基于本身硬件的优化也可以让它很快，比如用segmented Based状态
- 内存访问上的差异似乎更多是来自于cache miss，如果能命中，Unix和为内核的差距并不大，且miss的情况大多数是capacity misses问题，可能和Mach内核本身过于笨重有一定关联，比如有一些长期经常被调用的操作一直在跑，或者这些操作用的cache working sets比较大

名词特别需要解释
- 缓存工作集（Cache Working Set）​​：指​​程序在短时间内频繁访问的代码和数据​​，这部分内容会被缓存（Cache）存储以提高访问速度。

微内核需要processor-specific以提高自身易用性，IPC和线程开销的优化对于其很关键

感觉更多强调的是硬件能力的最大化，以及打破了微内核性能就是不行这一点固有认知
## Lec 3:
### In Class:
#### Exokernel
原本OS的一些抽象限制了app本身更多性能的提升

为了更好地把资源给app，为此提出了三个技术
- secure bindings
- visible resource revocation
- abort protocol

首先指出了general-purpose来让特权级系统软件来做系统抽象的劣势
- 过于general-purpose的实现会影响性能，尤其是程序行为比较固定的情况下（比如说数据库）
- 高抽象导致一些关键和底层信息被屏蔽，让应用程序不可见
- 高抽象导致某些fancy的功能也没有用了

LibOS是Exokernel为了让应用能够得到底层细节的一个重要尝试，同时Exokernel需要给LibOS足够的底层细节，和基本的安全保护

LibOS设计原则
- 尽可能直接把硬件资源和底层资源给出，但是并不直接去分配资源
- 参与资源分配的过程，能够去要硬件资源
- 对于物理资源需要有一些名字来去掌管，具体体现在某些数据结构的管理上
- 能够回收之前要过来的资源

具体技术
- Secure bindings：提供access time和bind time两个原语，在access time的时候显著减少原本可能需要的安全检查，从而提高性能，这是原意，但因此在bind time的时候，一定要做好检查。一些做法可能类似于软件实现的TLB
- multiplexing phyiscal memory: 已经由OS分配了权限的应用程序可以把自身的权限能级送出去，而送出去的这个过程就不需要内核来介入了
- multiplexing the network: 硬件上可以让某些应用专门走哪些stream，软件上可以以packet filters的模式来做
- visible resource revocation, or not: 各有千秋，但是visible确实更容易知道用户程序本身的需求
- abort protocol：出错了之后利用repossession exception来让LibOS来代劳，用户本身很难把这些情况描述清楚，因此这边添加了一个新的异常类型

优势
- 处理异常和系统调用都快了超级多
- 主要原因是避免了进入到内核数据结构中，少了一些TLB miss
- 感觉更多是软件实现上的性能提升，但是很显著
- 感觉更多是hardware-aware的设计

ASH
- 用户写好的，经过内核认证的，最后能够在内核这边跑起来的东西
- 我曹，这东西不就是eBPF吗
#### A fork() in the road
fork在现在是有点老，他确实有一些优势
- 编程范式很方便
- 很简洁
- 对于同步似乎也有一些裨益

fork的劣势
- 实际上一点都不简单，一大堆历史包袱
- 很暴力，对于用户程序来说可能摸不到核心，使用fork容易出现不可预知的中间态，尤其是IO的情况
- 并非是线程安全的，子进程很难完整地把父进程的情况搞过来，尤其是有多个竞态线程的情况下
- 很慢
- fork不适合scaling，按照直觉，想要scaling的话，应该避免何意味的共享，这也让fork更加适合宏内核，阻碍了原本的发展
- fork导致了内存上的overcommitment，让这种思潮影响了普通程序的开发

和deepseek的讨论
```
为什么在K42研究操作系统上移植 fork()会失败？

论文作者在K42上的亲身经历是论证 fork()危害系统创新的关键例证。K42的设计目标是成为一个​​高度可扩展、模块化、面向对象​​的OS，其核心哲学是​​分解状态和功能，避免集中化​​，以更好地适应大规模多处理器环境。
然而，为了实现与Linux的ABI兼容性，他们决定高效地实现 fork()。这个决定最终“绑架”了整个系统设计，导致了以下问题，这与您在TEE开发中遇到的困境异曲同工：
​​破坏模块化和反 locality​​：在K42中，进程的资源（内存区域、文件、文件描述符等）由不同的、可定制的对象实例管理。为了支持 fork()，​​每一种​​这样的对象类型都必须实现自己的 fork行为。这给那些原本只为特定计算任务优化的、根本不需要 fork的组件带来了巨大的、不必要的复杂性。

​​被迫引入“惰性”和集中化​​：为了缓解性能问题，他们不得不为所有对象实现类似写时复制的惰性复制机制。这不仅在每个对象内部增加了复杂性，还要求对象之间维护由 fork创建的层次关系。这直接违背了他们“限制共享和同步以提升局部性”的初衷。

​​损害可扩展性​​：尽管 fork协调是在库代码中完成的，但它需要与进程可能连接到的​​每一个​​服务器和对象进行通信。这引入了全局协调和同步，在NUMA系统中，fork还不得不访问父进程所在节点的内存，或在系统的竞争部分调度子进程，这与K42的核心可扩展性原则背道而驰。

​​结论​​：K42的教训是，fork()的语义——“复制一切”——与模块化、可扩展的OS设计根本冲突。试图高效地支持它，会迫使系统设计向传统的、 monolithic 的、状态集中的Unix模型靠拢，从而扼杀创新。

2. 为什么 fork()与单地址空间和异构硬件不兼容？

这部分论文的解释比较技术，我为您拆解一下：

为什么不能在单地址空间中使用？

​​根本矛盾​​：fork()的核心语义是创建一个与父进程​​隔离的、独立的​​新地址空间副本。而“单地址空间操作系统”的所有进程都共享同一个物理和虚拟地址空间，进程间的隔离是通过软件保护（如能力系统）而非硬件内存管理单元实现的。
​​无法实现​​：在一个单一的、共享的地址空间内，如何“复制”出一个隔离的副本？这从定义上就是不可能的。这就像要求在一张纸上复制出另一张独立的纸，而不允许使用第二张纸。
​​现实案例​​：论文提到，Drawbridge库OS可以在一个picoprocess（单地址空间）内实现多个Windows虚拟进程，因为它使用的是 CreateProcess模型，即​​加载新程序到地址空间的不同部分​​，然后创建一个新线程来执行。而基于 fork()的Unix模型则无法做到这一点，导致Unikernel和SGX enclave等环境要么不支持多进程，要么需要极其笨重和缓慢的模拟（如Graphene-SGX通过RPC在主机进程间复制内存，耗时可达数秒）。

为什么不能在异构硬件上使用？

​​状态复制的不完整性​​：fork()只复制CPU和内存的状态。但对于现代异构硬件（如GPU、DPDK智能网卡、FPGA加速器），进程在这些设备上也有大量的状态（如GPU内存、命令队列、网卡队列描述符、寄存器状态）。
​​OS无法透明处理​​：操作系统内核无法像复制内存页那样，透明、正确、原子地复制这些加速器上的复杂状态。当父进程 fork()时，子进程会继承一个“残缺”的硬件视图：
子进程可能无法访问父进程在GPU上分配的内存。
子进程尝试操作硬件队列可能导致错误或崩溃，因为硬件上下文并不属于它。
这会造成极大的混乱和不确定性。正如论文所引，GPU程序员对此问题已经困惑了十多年。
​​范式冲突​​：fork()的“复制”范式与异构计算的“进程独占和控制硬件”的范式相冲突。安全的方法是让进程显式地初始化和管理加速器资源，就像 posix_spawn或 CreateProcess那样，从一个干净的、已知的状态开始。
```

### Take Home
#### The Wrong Patient
就诊时出错后的多米诺效应，主要落实在解决方案上
- 标准化程序化的重要性
- 彼此之间沟通的必要，和积极参与

这篇工作感觉更多是方法论上的事情
#### Radiation Offers New Cures, and Ways to Do Harm
看起来就是讲安全事故和避免，本意和上一篇工作一样

#### The Task of the Referee
如何审稿
- 先给评分
- 总结文章
- 对文章的研究目标做评估
- 对文章的质量做评估，主要落实在方法论，精准度，技术，表达上

不好的文章
- 有些文章很烂，找几个致命错误就行了，没必要读完
- 不要人身攻击

主要从审稿人的视角，提供论文作者一些想法，应该怎么思考论文的写作

但是这篇文章的知识密度不高，随便看看就行
## Lec 4:
### In Class
#### The Rise of ``Worse is Better''
说白了就是大而全，小而快两种哲学的碰撞

作者更倾向Unix的简化设计，先像病毒一样完成一小部分功能，抢占先机，之后再fake it and make it
#### Therac-25
严重放疗事故
- 用软件检查代替原本独立的硬件安全检查，导致容易出错
- 有多处致命漏洞，存在竞态条件，未能同步
- 错误提示并不能让使用者理解
- 测试与质量管控不大好

建议
- 软件无故障是一种迷信，不应该新人
- 软件工程规范很重要
- 避免补丁类型修复，需要全面排查根源

可靠并不代表安全，尤其是这种敏感医疗器械

### Take Home
#### Synthesis Kernel
一个很有意思的内核
- 以threads作为基本单元
- 使用同一个地址空间，但是内核不让他们看到自己不应该看到的部分
- 用IO来传递threads之间的数据，文件和信息

减小内核的运行操作次数
- 使用基本内核元素来组织构建线程
- 切换线程减少了要存放的东西，不通过内核调度，而是自己构建了一个Round Robin序列来进行调度

阅读了Synthesis Kernel一文，感觉有一定的收获。其中的线程最小单元和SGX感觉是不谋而合的，我于是很好奇是否能够像Data Enclave工作一样，我们做一个Synthesised Enclave，负责给SGX做代码合成和性能加速
似乎这可能有点过于激进，仔细看了一下工作，其thread去减少syscall的情况，其实已经给很多bypass工作做烂了，比如hotcall，switchless等等。我们应该继续尝试去理解和搞清楚的是，这个内核合成代码的思想是否还有做的价值。以往安全圈子会避讳这些，但是如果TEE作为安全边界，是否可以避免类似的问题？

我本来觉得他的内核合成代码是一个很fancy的东西，但是实际上所使用的优化方式非常有限，不过确实可以让我们去尝试多在内核这边减少一下通信的路径

得有机会多去看看bypass相关的论文了
#### Multics
这篇工作主要讲了在multics中对于信息保护的一些准则和设计理念
- 保守的防御举措，与其考虑不让做什么，不如考虑允许做什么
- 对于任何object的访问，都要检查当前权限是否合适
- 对于设计本身，并不需要太多秘密
- 最小特权原则
- 设计原则不应该特别复杂，需要比较符合人类直觉

下面是主要的一些贡献
- 一个高可用性的，高扩展性的ACL机制前身对付文件系统
- 一个主要是把操作和某个用户绑定的问责机制，搭建多层次的认证架构（包括log等）
- 一个复杂，但是好用的内存保护模式，有一个东西叫做受保护子系统，只能通过特殊的位置进入。在进入时允许提权，出去后就恢复了原本的权利。（用户不信任子系统，但是系统信任子系统）

好像这边的内存部分很有意思，可以仔细看看，尤其是子系统！也许对我想要在机密计算场景下有点关联。

啊哈，这不就是enclave call吗。本质上就是受控接口调用！

然而对于这个segment descriptor，multics居然把他放到文件中持久化保存，其对应的安全防护居然是ACL。这可能就是最大的一个局限性吧。
## Lec 5
主要是看那本书，对应的论文我们顺便过一遍，这里的论文和书籍都非常经典，建议全部仔细读

#### Xen
大名鼎鼎的Xen项目，在Xen出现的时候，已经有了全虚拟化，但是性能不大好，也没法像Xen一样支持100个虚拟机

为了更好地提高系统的性能，Xen尝试multiplexes了物理资源，也避免了一些麻烦
- 内存管理可以直接去读系统硬件的页表，但是对于其修改得让hypervisor来
- 不是所有系统调用都需要下陷，除了页表异常以外，其他大部分还是代劳
- 裁剪了客户OS的硬件中断数目

Xen的CPU虚拟化快速处理
- system calls: 走快速路径，直接和cpu交互
- page fault: 这个没办法，只能由xen来代劳，因为CR2只有ring0能看


IO rings这个事情是比较优雅的解决方式

这个问题最后被硬件虚拟化解决，让page fault在EPT表中，不需要下陷到最高权限也能操作

#### Are Virtual Machine Monitors Microkernels Done Right?
认为本质上VMM管理多个虚拟机的方式，和微内核管理多个组件的方式存在异曲同工之妙

未来微内核的思想可以用来指导VMM去设计，去尝试让各个OS之间进行协作，这是一个比较有insights的文章

#### Disco
工作主要聚焦在如何在有可扩展共享内存的多处理器下，做出来一个高级玩意儿
- 本质上就是尝试给出一个观点，当新型硬件出来，我们应该怎样避免对于系统软件的大改，而比较轻易地达成我们的目标
- 这是一个很重要的问题，因为系统软件写起来有滞后性

本质上就是加一个抽象层，但是他加的特别好！

几大挑战
- trap and emulate性能的开销
- 代码和数据的冗余
- 高了一层抽象级，对于上层OS具体对于内存的使用缺乏更精细的管理能力
- 共享这件事很难

Disco提出的抽象和制作的技巧
- 硬件友好：善用NUMA硬件本身的特性，尽可能使用local备份
- 减少同步：设计共享内存和核间中断进行通信，尽量减少锁的使用次数
- vCPU：第一次建立了这个概念，并存储上下文信息
- 共享：transparently共享被复用的页
- 文件系统：copy-on-write disks，对于NUMA硬件友好

其他：
- 做测试做了很多breakdowns，非常精彩
- 使用多种测试证明其所能达到的多种特性，并加以解释，展现了设计系统的优势所在

精彩的反直觉
- 添加了一个虚拟层反而性能变的更好了
- 这个垫片相比于老旧的操作系统，通过全局优化和压榨硬件性能，非但没有损耗性能，反而在多机器的场景下展现了很好的性能提升，具体提升的部分主要是在同步层面上的事情，主要优化有两个，其一是lock free共享数据结构的设计，其二则是对于lock上的页，更多的任务降低了因为trap and emulate时cpu等待所遭到的罚时

#### The Turles Project
超好的一篇工作！系统梳理了嵌套虚拟化遇到的挑战，解决方案，以及性能优化方式

这篇工作特别有insights，无论是设计内容还是最后跑分，都很有价值，其所做的breakdown细致入微，发人深省，值得全部读完！