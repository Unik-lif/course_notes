其实本来是打算把`6.828`先行做完的，但是中途去搞`rCore`了，现在回来想把这玩意儿做完。

### ex1: syntax of assembly code.
`Intel`与`at&t`就语法上，最本质的区别其实还是顺序。前者的`source`数据一般都是在右边。如果要把`ebx`用`eax`来赋值，就`at&t`与`intel`来看分别是下面这样。
```
movl %eax, %ebx

mov ebx, eax
```

使用`ctrl + a x`来退出`qemu`
### ex2: 跟踪
这一部分希望我们通过`si`来简单跟踪一下后续的指令，主要做的事情有很多零碎并且似乎没有什么用的细节，它们是写死在`bios`之中的，与我们后续启动的操作系统关系并不是那么大。

我们可以看到一些`in`和`out`的端口读取操作。需要注意的是，如果我们只是想写一个操作系统，至少`bios`部分的东西，这是硬件厂商所提供的，我们可以不太管。
