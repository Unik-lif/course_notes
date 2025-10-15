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