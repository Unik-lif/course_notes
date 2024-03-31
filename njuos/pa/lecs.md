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