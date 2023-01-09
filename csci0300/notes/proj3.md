## Conceptual Questions
1. Come up with a real world analogy to explain caching to someone who does not know what a computer is.

借用Dan Garcia的叙述：我人住在圣芭芭拉，想要吃德州的蔬菜，在加州首府萨克拉门托有一个杂货店能够满足我的进货需求。这样，我想搞来蔬菜吃，就可以直接取萨克拉门托，而不必去德州。

2. What are the benefits of having a standard file I/O API provided by the operating system? How does this help programmers who might not know what hardware their programs will be running on?

对于高级语言开发者，他们不需要太了解底层的一些细节。设置文档完整、功能清楚的API会把工作大大简化。

3. Read about the catchphrase “Everything is a file” that is central to the design of UNIX-like operating systems (e.g., Linux, macOS, Android). Why might this complicate caching I/O?

因为全部的东西都会被抽象成文件，而各种各样的设备存在较大的差异，却要用同一的cache联系在一起。因此，驱动相关的I/O cache要考虑很多硬件参数细节，才能让执行效果相对更好一些。

