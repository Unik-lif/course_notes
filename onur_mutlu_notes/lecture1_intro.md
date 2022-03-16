## 体系结构当前四大方向：
1. 基础安全研究
2. 系统节能高效的体系结构
3. 低延迟和可预测的体系结构
4. 特殊计算用途的体系结构（如AI等方向
   
### Axiom
如果要得到最大能效和性能，需要有扩展的视野，打通计算机全栈。

we must take the expanded view

做好软硬件协同：

例子，Tesla和Google设计的特殊体系结构。

Google2017年发了一篇TPU的文章，可以去参考一下。
### 作为基础体系结构课
仅focus on ISA， 微体系结构，和logic部分。

#### CA是什么？

设计计算平台的科学与艺术。

在计算机栈的顶层和底层都有很丰富的课题去探讨，应该结合起来讨论。

### 目前一些相对有趣的研究现状

#### ex1
2019年： non-volatile main memory。intel optane persistent memory。相对于一般情况下DRAM内存，其有很大的优势。

在2009年，这个想法就有了。

#### ex2 Cerebras's wafer scale Engine
wafer with great ability for computing

在晶圆上做了一个很小的系统。

是算法和ca的结合。为了算法优化而制作了一个特殊晶片，当然，损耗的能源也很多。
#### ex3 
无需将数据移到DRAM芯片之外。直接在内存里进行处理，不用送给处理器了。

减少了访存的时间消耗。

在十年前也不可行。

#### ex4
三星公司针对机器学习应用程序制造了新的体系结构。

在芯片之中加上很多layer层，就和神经网络处理数据一样，总而言之现在的研究方向确实非常的广。

### Reliablity and security

#### Rowhammer 2014
One can predictably induce bit flips in commodity DRAM chips

first example of how a simple hardware failure mechanism can create a widespread system security vulnerability

通过比特的翻转来搞到root权限。

反复地读取、关闭，不停通电和断电，可以在DRAM chips中临近的行里产生扰动。

被google project zero证实能够在用户态翻转比特，绕过内存保护。

在安卓上，gpu上，network上，RAM上，DNN上，总而言之很多层面都有攻击例子。

#### 关于meltdown和spectre

project zero.

### Dream and, they will come

现在所需的计算性能越来越高，对体系结构也带了更多的机遇和挑战。

### Perspective

you can revolutionize the way computers are built if you understand both the hardware and the software.

#### Recommended book: the structure of scientific revolutions.

##  Assignments part:
### Reading
#### Cramming More Components onto Integrated Circuits
摩尔的这篇文章总结了集成电路的可靠性：通过阿波罗飞船火箭的例子，预见了现如今使用手机等设备的未来。

集成电路一个巨大的吸引力在于其较低的成本，对于简单电路来说，其在不同阶段均存在最小成本点。如下图所示：

![photo1](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-19%20211744.png)

摩尔认为在当前很长一段时间内，集成电路可以充分利用铺板时的空间，减少浪费，从而增加晶体管的密度。其认为这没有理论难度，只是一个工程上的问题。

根据目前这个density与年代变化的关系，摩尔做了下面这个图，做了一个大致的预测（但是没有文字信息）。

![photo2](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-19%20214455.png)

摩尔这篇杂谈的文字并没有特别值得让人留意的地方，除了这两张图表，大部分想法并没有特别突出。

（感觉有点被神化了）

#### chapter1 H&H：digital design using vhdl
数字系统流行的一个主要原因是其可以在有效避免噪音干扰的情况下做好信号处理等相关工作。在噪音信号$\epsilon$的值尚未达到足够大的情况下，其可被忽略。

因此，不同于模拟电路，数字系统将会在传输时省略掉这一误差，从而避免错误的累积，如下图所示：

![photo3](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-19%20224923.png)

数字系统达成这类功效的其中一个要点即为，让合法输出部分相比输入部分更加的狭窄。这样可以规避部分噪声影响，并让经过自身元件时受到的干扰影响更小。

因此，对于一个设备来说，其容忍错误的程度取决于以下值的长度大小：
$V_{NM}/(V_{1}-V_{0})$。
