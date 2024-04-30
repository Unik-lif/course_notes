## Chapter5
我们主要在这边对和 Lab 没有太大关系的代码进行分析，作为听课后或者听课前的随堂作业。
```c
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  uint64 scause = r_scause();

  // 翻阅手册，这边所指的是 external interrupts ,其中最高一位 bit 用于表明这个东西是 interrupt 类型
  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.

    // irq indicates which device interrupted.
    int irq = plic_claim();

    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      virtio_disk_intr();
    } else if(irq){
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000001L){
    // software interrupt from a machine-mode timer interrupt,
    // forwarded by timervec in kernelvec.S.

    if(cpuid() == 0){
      clockintr();
    }
    
    // acknowledge the software interrupt by clearing
    // the SSIP bit in sip.
    w_sip(r_sip() & ~2);

    return 2;
  } else {
    return 0;
  }
}
```
其中 plic_claim 的解析如下所示：本质上，我们可以把他当作硬件平台的中断控制器，可以用来存放异常发生时的中断现场，如修改部分状态寄存器，但是上下文信息的存储似乎交给了我们的软件来做？
```C
// ask the PLIC what interrupt we should serve.
int
plic_claim(void)
{
  // 确认当前的 cpu 是哪一个号
  int hart = cpuid();
  // 根据 CPU 号来寻找对应的 Hart 所对应的 irq 号
  int irq = *(uint32*)PLIC_SCLAIM(hart);
  return irq;
}

// tell the PLIC we've served this IRQ.
void
plic_complete(int irq)
{
  int hart = cpuid();
  *(uint32*)PLIC_SCLAIM(hart) = irq;
}
```
一些细微的差别： qemu 对于 PLIC 特定地址的设置与真实的 SiFive 手册存在一些细微的区别，我们在 qemu 这边还是主要感受其功能上的不同，其他的不要太计较。

