## PA0
### VIM usage
```
i1<ESC>q1yyp<C-a>q98@1
```
q1是记录这个名字叫1的宏，98@1是结束这个名为1的宏。

yyp是复制当前行，并向下粘贴，Ctrl A在Vim中，一般用于对光标所在的位置的数字进行递增操作。

下面的98是执行的次数。之后就重复之前的向下复制，并且递增的过程，执行98次，因此最后我们可以得到1-100这样的效果。

```
<C-v>24l4jd$p
```

### Unix哲学
这种东西确实好：
```
find . -name "*.[ch]" | xargs grep "#include" | sort | uniq
```

一些工具：
```
熟悉一些常用的命令行工具, 并强迫自己在日常操作中使用它们
文件管理 - cd, pwd, mkdir, rmdir, ls, cp, rm, mv, tar
文件检索 - cat, more, less, head, tail, file, find
输入输出控制 - 重定向, 管道, tee, xargs
文本处理 - vim, grep, awk, sed, sort, wc, uniq, cut, tr
正则表达式
系统监控 - jobs, ps, top, kill, free, dmesg, lsof
上述工具覆盖了程序员绝大部分的需求
可以先从简单的尝试开始, 用得多就记住了, 记不住就man
```
### git
对于git我目前仅仅知道一些粗暴的使用方法，应该抽空学习一下这些东西：

https://onlywei.github.io/explain-git-with-d3/

git commit --allow-empty这个很有意思

**PA0 Done**
## PA1
### ccache
通过RTFM可以得知，想让ccache生效，需要在路径中添加一个东西。如果想要让东西在当前目录生效，可以在这个目录中添加`.bashrc`文件，然后再`source`使之生效。

我们先使用`riscv32`来做吧，我的目的是先通关南大的相关课程，把基础知识补上。
### 寄存器的问题
当然可以没有，就是慢了很多，花更多时间取值

会让本来的一些local特性消失掉，失去了原来的多层次，常利用的特性，让性能早早达到瓶颈，性能的变化也会影响ISA之上编程模型的变化，基本把整个程序应用生态给毁灭了

失去了hardware-awareness
### 状态机绘制
```
// PC: instruction    | // label: statement
0: mov  r1, 0         |  pc0: r1 = 0;
1: mov  r2, 0         |  pc1: r2 = 0;
2: addi r2, r2, 1     |  pc2: r2 = r2 + 1;
3: add  r1, r1, r2    |  pc3: r1 = r1 + r2;
4: blt  r2, 100, 2    |  pc4: if (r2 < 100) goto pc2;   // branch if less than
5: jmp 5              |  pc5: goto pc5;
```
应该挺简单：
```
(0, x, x) -> (1, 0, x) -> (2, 0, 0) -> (3, 0, 1) -> (4, 1, 1) -> (5, 1, 1) -> (6, 1, 2) -> (7, 3, 2) -> (8, 3, 2) ... 
```
### 阅读代码
Kconfig用于系统的配置

后面跟着教程走就好了。
### RTFSC
教程里的步骤写的非常详实，对于新手很友好。

一些问题解答：

`cmd_c()`函数中`cpu_exec()`的传参为`-1`，这样使得我们的CPU能够一直跑下去，跑足够多下。因为是`uint64_t`类型，所以实际上最是最大的那个`64`位数。

根据C99的说法：`An example of undefined behavior is the behavior on integer overflow.`，但是这里并没有出现`overflow`，所以其实还好，主要还是数据类型设置的不错，就避免了这个问题。

程序从`main`开始似乎是由`libc`中决定的，在`bare metal`中这个就不一定了，比如先前撰写`rCore`时的经验，程序确实不是从`main`开始的。因此我感觉，能够在完成`main`后结束，也多少带着类似的概念。

`nemu`中如果没有引入`trap`或者中断的概念，可能这个程序就是跑不完的。

这里有几个用过，有几个没有试过，可以在之后具体写代码、读源码的过程中尝试。
```
为了帮助你更高效地RTFSC, 你最好通过RTFM和STFW多认识GDB的一些命令和操作, 比如:

单步执行进入你感兴趣的函数
单步执行跳过你不感兴趣的函数(例如库函数)
运行到函数末尾
打印变量或寄存器的值
扫描内存
查看调用栈
设置断点
设置监视点
```
应该只有 运行到函数末尾 这个是新的，大概跑一个 finish 就好了，看起来也不是特别麻烦。

小练习：
```
[src/monitor/monitor.c:35 welcome] Exercise: Please remove me in the source code and compile NEMU again.
(nemu) q
make: *** [/home/link/PA/ics2023/nemu/scripts/native.mk:38: run] Error 1
```
使用`q`的时候返回为-1，之后就会`return`出去，回到`main`函数这边。
```C
int main(int argc, char *argv[]) {
  /* Initialize the monitor. */
#ifdef CONFIG_TARGET_AM
  am_init_monitor();
#else
  init_monitor(argc, argv);
#endif

  /* Start engine. */
  engine_start();

  return is_exit_status_bad();
}
```
调用了`is_exit_status_bad`函数，必坏无疑。改一下返回值为`0`就好了。