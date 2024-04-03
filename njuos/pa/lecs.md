## Lec3:
如何代码本身不依赖注释，别人就能看懂？

举一个例子：
```C
void (*signal (int sig, void (*func)(int)))(int)
```
改写为：
```C
typedef void (*sighandler_t)(int);
sighandler_t signal(int, sighandler_t);
```
显著提高了可读性。

黑魔法：
```C
#define FORALL_REGS(_)  _(X) _(Y) _(Z) // 添加Z.
#define LOGIC           X1 = (!X && Y) || (X && !Y);\
                        Y1 = !Y; \
                        Z1 = !Z
```
来自宏里头的简单作业：

尝试查阅一下`##`和`#`是什么意思？

维持一个框架，然后对于代码的修改我们只要维持在`LOGIC`部分的局部就足够了。

特殊指针类型：
```C
#include <stdint.h>
#define NREG 4
#define NMEM 16
typedef uint8_t u8;
u8 pc = 0, R[NREG], M[NMEM] = { ... };
```
用来避免跨平台字长分配出现不同的一个设置。`unsigned` 类型可以避免潜在的 `UB`，在 `gcc` 中似乎有一个 `-fwrapv` 可以玩，强制有符号整数能够溢出为 `wraparound`。

提升代码的质量：把寄存器的名字嵌入进去。
```C
enum { RA, R1, ..., PC }; // for registers.
u8 R[] = {
    [RA] = 0,
    [R1] = 0,
    ...
    [PC] = init_pc,
};
// equal to 
// u8 R[] = {0, 0, 0,0 ,0, ...., init_pc};
// but more readable.

#define pc (R[PC]) // take pc as a part of the registers. get pc from R arrays.
#define NREG (sizeof(R) / sizeof(u8))
```

减少软件中的隐藏`dependencies`，降低代码的耦合程度。
```C
#define NREG 5 // 假设 max {RA, RB, ..., PC} == {NREG - 1}

#define NREG (sizeof(R) / sizeof(u8))

#define NREG (sizeof(R) / sizeof(R[0]))

// An even better way.
// I've seen this kinda in Hyperenclave-driver.
// NREG 可以自动获得到一个值，正好对应前面寄存器数目的总和，RA的默认值则是0。
enum {RA, ..., PC, NREG}
```
指令集解析无敌写法：
```C
typedef union inst {
    struct { u8 rs: 2, rt:2, op: 4;} rtype;
    struct { u8 addr: 4, op: 4;} mtype;
} inst_t;

#define RTYPE(i) u8 rt = (i)->rtype.rt, rs = (i)->rtype.rs;
#define MTYPE(i) u8 addr = (i)->mtype.addr;

void idex() {
    inst_t *cur = (inst_t *)&M[pc];
    switch (cur->rtype.op) {
        case 0b0000: { RTYPE(cur); R[rt] = R[rs]; pc++; break; }
        case 0b0001: { RTYPE(cur); R[rt] += R[rs]; pc++; break; }
        case 0b1110: { MTYPE(cur); R[RA] = M[addr]; pc++; break; }
        case 0b1111: { MTYPE(cur); M[addr] = R[RA]; pc++; break; }
        default: panic("invalid instruction at PC = %x", pc);
    }
}
```
## Lec4:
`Git`和`Github`与代码仓库，尽可能生成一个`diverse`的测试生成脚本。

怎么写项目的`Makefile`。

查询`github`上对于论文的精选资源。

`github`工作区：我们所能看到的文件

`github`的暂存区：`Stage`，我们希望通过`commit`上传的一小部分文件

这边的方法论在于：我没有必要对所有的文件进行`commit`。

`git merge`：合并多人协作时的场景。

visualizing git concepts with D3

白名单：`.gitignore`文件

`.git`文件中包含着`OJ`想要知道的一切。

`PA`究竟在做什么？在做一个`AM`，即`Abstract Machine`。

`vim`的插件配置：
```
nerdtree, themes, ctags(代码跳转)
```

其他一些听起来比较有意思的工具：
```
difftest，sdb
```
## Lec5: Nemu框架选讲
`Makefile`是一个`declarative`的代码，描述了构建目标之间的依赖关系和更新方法。

尝试去读懂`Makfile`内部的语法，了解项目构建的意思。

以管道的方式拷贝到`vim`中。
```
make -nB \
    | grep -ve ''
    | vim -
```

`ld`对应的链接器和`gcc`使用的链接器是不一样的，前者是默认配置，还需要把很多动态库比如`libc.a`等链接进来，后面的只需要加上我们自己写的头文件就行。

现代文档编写方式：`docs as code`

`auto_config`

`gcc -E`预编译看一下宏是否被定义了。

`static`关键字，可以用于让某个函数或者变量仅在某个文件中可见，就能做到避免某个编译单元有两份可见的函数定义。

`static inline`关键字，告诉编译器，建议以内联的方式来使用它，不把它当做一个真正的函数，典型的以空间换时间的函数，一般希望这个函数很小。

尽可能要给宏加上括号，避免一些预编译时复制粘贴上的影响：
```
#define assert(cond) if (!(cond)) panic(...);

if (...) assert(0);
else ...
```
=> else将会基于文法匹配到`panic`，就近匹配。

如何高效地使用`Vim`编辑器来进行跳转？需要频繁的进行代码跳转。

难以调试的原因：
- 两层软件抽象，monitor 与 gdb
- 

## Lec6：数据的机器级别表述
计算机是怎么理解零和一的。

C语言有很多基于比特的操作，位运算是用电路最为容易实现的逻辑。

bit级别的表示，如果能够操作的对象刚好每个bit是独立的，那么一个位运算就涉及多条指令的同时执行。在这种情况下，如果能够有一个`bitset`来进行管理，或许在性能上会有一些提高。

我们可以用`5`，即`0101`来表示`{0, 2}`这样的一个集合，那么如果要判断一个`x`数据是否在这个集合，我们直接这么做就好：
$$
(S >> x) & 1
$$

使用位运算来计算`|S|`看起来还是挺好玩的。

课上有一种奇妙的`LOG2(x)`求值方式，非常牛逼，搁这空间换时间打表呢。

推荐书籍：hacker's delight.

Undefined Behavior(UB)

通常C对于UB的行为不做多少约束，只要能够把电脑搞炸了就行了。出现了UB行为，编译器会有自己解读的一些偏好，C语言并没有要求编译器怎么实现UB时的异常处理。

在这边提到了先前南大写的用来判断浮点数`IEEE754`标准时表达疏密度的一个检测程序，因为该疏密度的存在，有很多数学库都会频繁做归一化处理，否则可能表达误差可以达到整数2这个级别。

浮点数的精度会有很有意思的问题：比如二次方程求根，做一个接近的公式反而要比原来的公式更加精确。

人物的穿模可能会和浮点数精度没做好有一定联系。