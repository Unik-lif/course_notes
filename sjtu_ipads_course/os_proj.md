## 实验记录
### Lab0
快速熟悉一下arm的汇编，当然还是有很多没有看懂

前几个Phase都很简单，感觉从Phase 4开始需要看一下objdump的结构想办法

#### Phase 4
最后会比较isggstsvkt这个字符串和输入的已经经过变换的字符串是否相同

所以我们得看懂对应的逆操作，然后输入可以最后加密得到这个值的字符串

```
0000000000400964 <encrypt_method2>:
  400964:	7100003f 	cmp	w1, #0x0
  400968:	540003cd 	b.le	4009e0 <encrypt_method2+0x7c>
  40096c:	a9bd7bfd 	stp	x29, x30, [sp, #-48]!
  400970:	910003fd 	mov	x29, sp
  400974:	a90153f3 	stp	x19, x20, [sp, #16]
  400978:	a9025bf5 	stp	x21, x22, [sp, #32]
  40097c:	aa0003f3 	mov	x19, x0
  400980:	8b21c015 	add	x21, x0, w1, sxtw
  400984:	90000516 	adrp	x22, 4a0000 <_GLOBAL_OFFSET_TABLE_+0x470>
  400988:	910162d6 	add	x22, x22, #0x58
  40098c:	14000009 	b	4009b0 <encrypt_method2+0x4c>
  400990:	39400281 	ldrb	w1, [x20]
  400994:	f94006c0 	ldr	x0, [x22, #8]
  400998:	8b010000 	add	x0, x0, x1
  40099c:	3859f000 	ldurb	w0, [x0, #-97]
  4009a0:	39000280 	strb	w0, [x20]
  4009a4:	91000673 	add	x19, x19, #0x1
  4009a8:	eb15027f 	cmp	x19, x21
  4009ac:	54000120 	b.eq	4009d0 <encrypt_method2+0x6c>  // b.none
  4009b0:	aa1303f4 	mov	x20, x19
  4009b4:	39400260 	ldrb	w0, [x19]
  4009b8:	51018400 	sub	w0, w0, #0x61
  4009bc:	12001c00 	and	w0, w0, #0xff
  4009c0:	7100641f 	cmp	w0, #0x19
  4009c4:	54fffe69 	b.ls	400990 <encrypt_method2+0x2c>  // b.plast
  4009c8:	9400004b 	bl	400af4 <explode>
  4009cc:	17fffff1 	b	400990 <encrypt_method2+0x2c>
  4009d0:	a94153f3 	ldp	x19, x20, [sp, #16]
  4009d4:	a9425bf5 	ldp	x21, x22, [sp, #32]
  4009d8:	a8c37bfd 	ldp	x29, x30, [sp], #48
  4009dc:	d65f03c0 	ret
  4009e0:	d65f03c0 	ret
```

encrypt_method2对应的是找字典，这个地方查找的字典是
```
root@2ef476682274:/OS-Course-Lab/Lab0# objdump -s --start-address=0x4647f0 --stop-address=0x464810 bomb

bomb:     file format elf64-little

Contents of section .rodata:
 4647f0 71776572 74797569 6f706173 64666768  qwertyuiopasdfgh
 464800 6a6b6c7a 78637662 6e6d0000 00000000  jklzxcvbnm......
```
这个是从符号表中确认得到的，挺有意思的！

那么对应的我们的isggstsvkt字符串对应的位置分别是
```
7 11 14 14 11 4 11 22 17 4
hloolelwre
```

那么似乎可以猜了，很有可能一阶段的就是helloworld，试试看


不对，没有d，我们替换一下，结果发现还真是这个结果
```
00000000004008c8 <encrypt_method1>:
  4008c8:	a9be7bfd 	stp	x29, x30, [sp, #-32]!
  4008cc:	910003fd 	mov	x29, sp
  4008d0:	910043e2 	add	x2, sp, #0x10
  4008d4:	3821c85f 	strb	wzr, [x2, w1, sxtw]
  4008d8:	0b417c23 	add	w3, w1, w1, lsr #31
  4008dc:	13017c63 	asr	w3, w3, #1
  4008e0:	7100043f 	cmp	w1, #0x1
  4008e4:	540003cd 	b.le	40095c <encrypt_method1+0x94>
  4008e8:	aa0203e4 	mov	x4, x2
  4008ec:	d2800002 	mov	x2, #0x0                   	// #0
  4008f0:	d37ff845 	lsl	x5, x2, #1
  4008f4:	38656805 	ldrb	w5, [x0, x5]
  4008f8:	38001485 	strb	w5, [x4], #1
  4008fc:	91000442 	add	x2, x2, #0x1
  400900:	6b02007f 	cmp	w3, w2
  400904:	54ffff6c 	b.gt	4008f0 <encrypt_method1+0x28>
  400908:	7100007f 	cmp	w3, #0x0
  40090c:	1a9fc465 	csinc	w5, w3, wzr, gt
  400910:	6b05003f 	cmp	w1, w5
  400914:	540001cd 	b.le	40094c <encrypt_method1+0x84>
  400918:	4b050024 	sub	w4, w1, w5
  40091c:	4b0300a2 	sub	w2, w5, w3
  400920:	8b22c402 	add	x2, x0, w2, sxtw #1
  400924:	91000442 	add	x2, x2, #0x1
  400928:	d2800001 	mov	x1, #0x0                   	// #0
  40092c:	910043e3 	add	x3, sp, #0x10
  400930:	8b25c065 	add	x5, x3, w5, sxtw
  400934:	d37ff823 	lsl	x3, x1, #1
  400938:	38636843 	ldrb	w3, [x2, x3]
  40093c:	382168a3 	strb	w3, [x5, x1]
  400940:	91000421 	add	x1, x1, #0x1
  400944:	eb04003f 	cmp	x1, x4
  400948:	54ffff61 	b.ne	400934 <encrypt_method1+0x6c>  // b.any
  40094c:	910043e1 	add	x1, sp, #0x10
  400950:	940084dc 	bl	421cc0 <strcpy>
  400954:	a8c27bfd 	ldp	x29, x30, [sp], #32
  400958:	d65f03c0 	ret
  40095c:	52800005 	mov	w5, #0x0                   	// #0
  400960:	17ffffec 	b	400910 <encrypt_method1+0x48>
```
这边可以看出来是做了打乱工作，奇数位和偶数位做了混淆

#### Phase 5
```
(gdb) x/44x $x1
0x4a0070 <search_tree>: 0x00000031      0x00000000      0x004a0088      0x00000000
0x4a0080 <search_tree+16>:      0x004a00a0      0x00000000      0x00000014      0x00000000
0x4a0090 <search_tree+32>:      0x004a00b8      0x00000000      0x004a00d0      0x00000000
0x4a00a0 <search_tree+48>:      0x00000058      0x00000000      0x004a00e8      0x00000000
0x4a00b0 <search_tree+64>:      0x004a0100      0x00000000      0x00000003      0x00000000
0x4a00c0 <search_tree+80>:      0x00000000      0x00000000      0x00000000      0x00000000
0x4a00d0 <search_tree+96>:      0x00000025      0x00000000      0x00000000      0x00000000
0x4a00e0 <search_tree+112>:     0x00000000      0x00000000      0x00000037      0x00000000
0x4a00f0 <search_tree+128>:     0x00000000      0x00000000      0x00000000      0x00000000
0x4a0100 <search_tree+144>:     0x0000005b      0x00000000      0x00000000      0x00000000
0x4a0110 <search_tree+160>:     0x00000000      0x00000000      0x00464a68      0x00000000
```
输入的值的要求，假设我们一开始输入的是5，用gdb si单步调试，这个过程就会变得非常清楚
- 不可以是0x31，这个可能是树的节点
- 如果比0x31小，则找到0x004a0088      0x00000000这个地址，放在$w1中
- 这个地址作为新的位置，找到节点中对应的14这个值
- 递归地往下去做，14还是太大，找到更小的节点，对应0x4a00b8
- 这个值找到的是3，我们从地址上可以找到
- 这个时候确实比3要大了，结果再去索引，不知为什么去访问了c8位置，结果x1加载到了0
- cbz发现是0，跳转到0x400ab8，然后会设置返回值为0
- 跳转到0x400ab4，此时w0值为1，又跳转回来，之后递归跳出，在w0中记录对应的递归深度，可能就是递归深度

最后退出的时候，会看一下返回值3，如果不是3，则炸

我们这边5一共比较了两次，进入0x3分支，但是比3大，是1左移两次，得到值为4
- 用25进行比较，进入0x25分支，但是比0x25小，一开始是0，然后是1左移动1次，得到值为2

树的结构是更小则跳8，更大则跳16，可能对应的数据结构式
```C
struct tree {
    unsigned long value,
    struct tree * smaller,
    struct tree * bigger,  
};
```
因此我们可以得到树的结构
```
       0x31
    /        \
  0x14       0x58
  /  \       /  \
0x3 0x25   0x37 0x5b

  4  2 6  1  5  3         
```
如果要得到3这个值，我们可以快速去跑来验证


b *0x400adc

尝试了一下，输入90就能得到3，因此最后的答案是
```
2022
The Network as a System and as a System Component.
1 1 5 9 17 29 49 81
3 3
helloworle
90
```
没有特别困难，做了大概几个小时就搞定了，不过真的挺好玩的，感觉自己调gdb有进步，而且arm的汇编对我来说确实是陌生的

```
root@2ef476682274:/OS-Course-Lab/Lab0# make grade
Grading lab 0 ...(may take 10 seconds)
===========================================
Phase 1: 15/15
Phase 2: 15/15
Phase 3: 15/15
Phase 4: 15/15
Phase 5: 20/20
All phases defused: 20/20
Score: 100/100
==========================================
```