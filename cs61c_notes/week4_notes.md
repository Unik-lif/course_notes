## Lecture:
something interesting:

the D-cache and L-cache(consider the structural hazard in pipeline of fetch the data and instruction from the memory at the same time)

superscalar processor.

## Proj3 done.
Some interesting things:

1. I can still debug by connecting what I need to the output registers.
2. Some subtle fault will counts when the circuits become big.
3. Connecting wires and circuits in Logisim is handy when the scale is small, but tedious when the scale is large, especially for the register files.

和本科时我写指令的经历不太一样，cs61c这边对指令做了很多的归结（比如把他们排布在一起进行研究，很大程度上方便了鄙人电路的搭建，等反应过来时，已经支持四十来条指令了），告诉了我写个电路的方法最好是全面而系统的，而不是针对功能不停地叠加。

这清楚告诉我一个精简且精妙的指令集能够给后续工作的落地带来多大的好处。

我有幸曾经对照着MIPS的使用手册一条条地理解相关的指令功能，然后敲verilog以实现一个单周期处理器。一开始看到RISCV似乎没有类似的、针对每条指令的详细手册，但是随着工程的推进我才发现把不同指令以类型进行绑定对照起来研究的巨大好处，他使得我们的世界观不再是零碎的，而是倾向于形成一个整体。

此外，tunnel真的是Logisim里头最棒的东西。

parta很简单没什么好提及的（就是写寄存器堆太折磨了），做这个实验partb花了两天，debug了两天，刚开始debug的时候确实感觉蛮绝望的哈哈哈。

不过确实学到很多就是了。谢谢伯克利的TAs和Profs哈哈。