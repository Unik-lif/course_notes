### Critically learn from design

推荐的课外阅读
#### You and Your Research
#### Meltdown & spectre
exploit speculative execution

disable speculative execution sometimes

mutlu讲到intel和部分研究者提供的方案是利用compiler的方式，同时也要设置部分的界限，用来时不时的隔绝掉乱序执行。当然，解决方法不一而足，而确定何时设置speculative disable指令同样也是比较复杂的工作。

因此，mutlu认为要打开计算机的抽象层界限，知道各层的工作机理，这需要对计算机体系有很深的认识。

think critically and think broadly

#### Rowhammer
disturbance

从2010年制造的芯片开始出现了这样的问题。

Dram cells are too close to each other.

是由于内存优化技术出现的物理问题。

google对这个工作做了优化。总之，就是不停刷新出现的结果呗。

2020年USENIX的顶会做的工作成功误导了深度学习网络。

如何进行防御？

potential solutions:
1. make better dram chips
2. refresh frequently
3. sophisticated error correction
4. access counters

拜占庭失败：rowhammer制造

打破了既定的假设条件。

同样也是打破抽象层面的一个攻击手段。

### Reading Assignment

#### chapter1 yale patt ICS

强调抽象的作用性：和出租车司机聊天，告诉他自己去机场而不说自己拟定的详细路线。

但是patt也指出，在细节层面无法正确工作时，要有能力去进行解包，脱离抽象层面去研究相应的组件。

这一章patt提出了两个核心观点：
1. 所有的计算机都能完成同样的任务，（在提供足量的资源时），只不过时间、效率、功耗可能会有不同。
2. 以计算机的思维、语言去理解计算机。

和Digital书上的作者一样，作者也表明了模电相对于数电的劣势。最早的数电机器其实就像是以往使用的古典加法器，对于不同器件，你只要告诉机器怎么做，接下来就是他们的事情。

最早的思想源自图灵的机器，利用黑盒这一抽象封装好，之后就直接用来满足我们的任务需求即可。

在图灵的文章中描述的广义图灵机能够和使用内存的计算机完成事实上一样的工作，在这两者的例子中，均是提供所需要的数，然后再算出值罢了。

因此，观点一可以得到佐证，因为他们均和广义图灵机等价。

很有意思的一点，原来onur mutlu的关于narrow view和broad view是从yale patt老师那边继承过来的。这边同样出现了计算机栈。

