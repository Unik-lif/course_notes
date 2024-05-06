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

对于 UART 16550 更好的手册资料：https://uart16550.readthedocs.io/_/downloads/en/latest/pdf/

下面我们对 uartinit 代码进行分析，先对于 line control register 做了相关设置，用来让我们能够配置带宽等信息。之后要想办法开启 FIFO ，以及让 UART 能够面对 transmit 以及 receive interrupts 时能够接受中断请求。
```C
void
uartinit(void)
{
  // disable interrupts.
  WriteReg(IER, 0x00);

  // special mode to set baud rate.
  // The internal baud rate counter latch enable
  WriteReg(LCR, LCR_BAUD_LATCH);

  // LSB for baud rate of 38.4K.
  WriteReg(0, 0x03);

  // MSB for baud rate of 38.4K.
  WriteReg(1, 0x00);

  // 到这边，针对 BAUD_DIVISOR 的设置似乎结束了，leave set-baud mode.
  // leave set-baud mode,
  // and set word length to 8 bits, no parity.
  WriteReg(LCR, LCR_EIGHT_BITS);

  // reset and enable FIFOs.
  /*
  FCR BIT-0:
  0=Disable the transmit and receive FIFO.
  1=Enable the transmit and receive FIFO. This bit should be enabled before setting the FIFO trigger levels.

  FCR BIT-1:
  0=no change.
  1=Clears the content of the receive FIFO and resets its counter logic to 0 (the receive shift register is not cleared or altered). This bit will return to zero after clearing the FIFOs.

  FCR BIT-2:
  0=no change.
  1=Clears the contents of the transmit FIFO and resets its counter logic to 0 (the transmit shift register is not cleared or altered). This bit will return to zero after clearing the FIFOs.
  */
  WriteReg(FCR, FCR_FIFO_ENABLE | FCR_FIFO_CLEAR);

  // enable transmit and receive interrupts.
  WriteReg(IER, IER_TX_ENABLE | IER_RX_ENABLE);

  initlock(&uart_tx_lock, "uart");
}
```
在初始化好 uart 之后，我们继续看 consoleinit 函数：
```C
void
consoleinit(void)
{
  initlock(&cons.lock, "cons");

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  // devsw 是对设备做的软件抽象，其中定义了driver所能进行的 read 与 write 的方式
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```
对于 console 的访问，文件头是在这边获取的：我们仔细分析一下 open 最后产生的 fd 的可能形态。
```C
// init: The initial user-level program

#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/spinlock.h"
#include "kernel/sleeplock.h"
#include "kernel/fs.h"
#include "kernel/file.h"
#include "user/user.h"
#include "kernel/fcntl.h"

char *argv[] = { "sh", 0 };

int
main(void)
{
  int pid, wpid;

  if(open("console", O_RDWR) < 0){
    mknod("console", CONSOLE, 0);
    open("console", O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf("init: starting sh\n");
    pid = fork();
    if(pid < 0){
      printf("init: fork failed\n");
      exit(1);
    }
    if(pid == 0){
      exec("sh", argv);
      printf("init: exec sh failed\n");
      exit(1);
    }

    for(;;){
      // this call to wait() returns if the shell exits,
      // or if a parentless process exits.
      wpid = wait((int *) 0);
      if(wpid == pid){
        // the shell exited; restart it.
        break;
      } else if(wpid < 0){
        printf("init: wait returned an error\n");
        exit(1);
      } else {
        // it was a parentless process; do nothing.
      }
    }
  }
}
```
我们尚没有学习到文件系统，目前尚且不知道为什么 console 是 device 类型的文件，我们目前姑且记住这一点。既然是 device 类型，那么我们之后在 open 和 read 的时候，会进入到特殊的被定义好的函数 consoleread 之中，这一点并不难看出。
```C
// Read from file f.
// addr is a user virtual address.
int
fileread(struct file *f, uint64 addr, int n)
{
  int r = 0;

  if(f->readable == 0)
    return -1;

  if(f->type == FD_PIPE){
    r = piperead(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    // 写入到 1 号文件描述符，我们在先前已经 init.c 函数中通过 dup 设置， 1 号其实就是 stdout
    r = devsw[f->major].read(1, addr, n);
  } else if(f->type == FD_INODE){
    ilock(f->ip);
    if((r = readi(f->ip, 1, addr, f->off, n)) > 0)
      f->off += r;
    iunlock(f->ip);
  } else {
    panic("fileread");
  }

  return r;
}
```
在这边，控制流显然会进入到第二个 else if 位置，我们因此可以调用 consoleread 函数：
```C
struct {
  struct spinlock lock;
  
  // input
#define INPUT_BUF 128
  char buf[INPUT_BUF];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} cons;

//
// user read()s from the console go here.
// copy (up to) a whole input line to dst.
// user_dist indicates whether dst is a user
// or kernel address.
//
int
consoleread(int user_dst, uint64 dst, int n)
{
  uint target;
  int c;
  char cbuf;

  // 目标是读取 n 个字符到 dst 位置上， user_dst 是文件描述符号
  // 这边把驱动也当作了文件来用，经典 Unix 哲学
  target = n;
  // 对于 console 的操作，一开始先上个锁
  acquire(&cons.lock);
  while(n > 0){
    // wait until interrupt handler has put some
    // input into cons.buffer.
    // 等待 read index 与 write index 的值不一样时再跳出？即我们确实已经有了一些输入进入到 buffer 内部
    // 在这边已经完成了对于 console 字符的输入工作了，接下来应该是解析
    // 在前面的 consoleintr 会把 cons.w 的值做更新，需要一个一个字符进行解析
    while(cons.r == cons.w){
      if(myproc()->killed){
        release(&cons.lock);
        return -1;
      }
      // 先小睡一会儿，我们不管这边的实现
      // 将会在未来的 consoleintr 函数中被唤醒
      sleep(&cons.r, &cons.lock);
    }
    // 解析刚读进来的字符
    c = cons.buf[cons.r++ % INPUT_BUF];

    // #define C(x)  ((x)-'@')  // Control-x 我们看一下 ASCII 表格这件事情就会变得非常显然
    if(c == C('D')){  // end-of-file
      if(n < target){
        // Save ^D for next time, to make sure
        // caller gets a 0-byte result.
        cons.r--;
      }
      break;
    }

    // copy the input byte to the user-space buffer.
    // 现在从内核态的 dst 位置读取到 user_dst 之中，这样 shell 似乎就能拿来用了？
    cbuf = c;
    if(either_copyout(user_dst, dst, &cbuf, 1) == -1)
      break;

    dst++;
    --n;

    if(c == '\n'){
      // a whole line has arrived, return to
      // the user-level read().
      break;
    }
  }
  // 对于 console 的 read 操作也已结束，毕竟我们用了 rw 以及 buffer 等物件
  release(&cons.lock);

  // 从返回值可以判断，是否过早 EOF 了
  return target - n;
}
```
那么这些字符是怎么送进来的？这边我们需要重新来看一下 PLIC 设备的一些细节：https://github.com/riscv/riscv-plic-spec/releases/tag/1.0.0

对于 PLIC 设备的硬件，可以参考这份资料，不过未来有空再看吧，我现在不是太关心过于硬件的事情，比如具体是哪一个号对应 UART0 ，我似乎没有找到相应的官方资料。
```C
// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from trap.c.
void
uartintr(void)
{
  // read and process incoming characters.
  while(1){
    // 先从 uart 设备读取到对应的字符
    int c = uartgetc();
    if(c == -1)
      break;
    // 之后想办法将其传输到 console 设备上
    consoleintr(c);
  }

  // send buffered characters.
  acquire(&uart_tx_lock);
  uartstart();
  release(&uart_tx_lock);
}

// read one input character from the UART.
// return -1 if none is waiting.
int
uartgetc(void)
{
  /*
    LSR BIT 0:
    0 = no data in receive holding register or FIFO.
    1 = data has been receive and saved in the receive holding register or FIFO.
  */
  if(ReadReg(LSR) & 0x01){
    // input data is ready.
    // RHR: receive holding register.
    return ReadReg(RHR);
  } else {
    return -1;
  }
}

//
// the console input interrupt handler.
// uartintr() calls this for input character.
// do erase/kill processing, append to cons.buf,
// wake up consoleread() if a whole line has arrived.
//
void
consoleintr(int c)
{
  // 需要对 console 做一些操作了，需要加上锁
  // 我们暂时不考虑其他的一些特殊情况，仅仅考虑一般的输入字符
  acquire(&cons.lock);

  switch(c){
  case C('P'):  // Print process list.
    procdump();
    break;
  case C('U'):  // Kill line.
    while(cons.e != cons.w &&
          cons.buf[(cons.e-1) % INPUT_BUF] != '\n'){
      cons.e--;
      consputc(BACKSPACE);
    }
    break;
  case C('H'): // Backspace
  case '\x7f':
    if(cons.e != cons.w){
      cons.e--;
      consputc(BACKSPACE);
    }
    break;
  default:
    // c != 0 并且 cons.e - cons.r 小于 INPUT_BUF ，即 INPUT_BUF 还没有用尽？
    if(c != 0 && cons.e-cons.r < INPUT_BUF){
      c = (c == '\r') ? '\n' : c;

      // echo back to the user.
      consputc(c);

      // store for consumption by consoleread().
      // 把 c 字符提前存放到 buf 之中， cons.e 专门用于记录从 UART 写入到 console buffer 的字符串出现的位置
      // 在之后， consoleread 就有办法从 buf 中读取出这个字符
      // 这个操作将会一直进行下去，即 buffer 会不断增长，直到有回车或者 eof 键出现
      cons.buf[cons.e++ % INPUT_BUF] = c;

      if(c == '\n' || c == C('D') || cons.e == cons.r+INPUT_BUF){
        // wake up consoleread() if a whole line (or end-of-file)
        // has arrived.
        cons.w = cons.e;
        wakeup(&cons.r);
      }
    }
    break;
  }
  
  release(&cons.lock);
}
```
同步，事先已经输出在屏幕上了，看的很清晰。
```C
//
// send one character to the uart.
// called by printf, and to echo input characters,
// but not from write().
//
void
consputc(int c)
{
  if(c == BACKSPACE){
    // if the user typed backspace, overwrite with a space.
    uartputc_sync('\b'); uartputc_sync(' '); uartputc_sync('\b');
  } else {
    uartputc_sync(c);
  }
}

// alternate version of uartputc() that doesn't 
// use interrupts, for use by kernel printf() and
// to echo characters. it spins waiting for the uart's
// output register to be empty.
void
uartputc_sync(int c)
{
  // push_off 和 pop_off 可以简单地当作一个锁
  push_off();

  if(panicked){
    for(;;)
      ;
  }

  // wait for Transmit Holding Empty to be set in LSR.
  // 一直等待，等待到 LSR 寄存器的 TX bit 终于是空的，那就说明 THR 寄存器现在可以用来操作了
  /*
    LSR BIT 5:
    0 = transmit holding register is full. 16550 will not accept any data for transmission.
    1 = transmitter hold register (or FIFO) is empty. CPU can load the next character.
  */
  while((ReadReg(LSR) & LSR_TX_IDLE) == 0)
    ;
  WriteReg(THR, c);

  pop_off();
}
```
我们再看到 write ，其实也是如出一辙的，我们看一下：
```C
//
// user write()s to the console go here.
//
int
consolewrite(int user_src, uint64 src, int n)
{
  int i;

  acquire(&cons.lock);
  for(i = 0; i < n; i++){
    char c;
    // 从 user_src 中写入到 c 中， user_src 是在 consoleread 时读进来的
    if(either_copyin(&c, user_src, src+i, 1) == -1)
      break;
    uartputc(c);
  }
  release(&cons.lock);

  return i;
}
```
继续观察函数 uartputc ,如下所示：
```C
// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }

  while(1){
    if(((uart_tx_w + 1) % UART_TX_BUF_SIZE) == uart_tx_r){
      // buffer is full.
      // wait for uartstart() to open up space in the buffer.
      sleep(&uart_tx_r, &uart_tx_lock);
    } else {
      uart_tx_buf[uart_tx_w] = c;
      uart_tx_w = (uart_tx_w + 1) % UART_TX_BUF_SIZE;
      uartstart();
      release(&uart_tx_lock);
      return;
    }
  }
}

// if the UART is idle, and a character is waiting
// in the transmit buffer, send it.
// caller must hold uart_tx_lock.
// called from both the top- and bottom-half.
void
uartstart()
{
  while(1){
    if(uart_tx_w == uart_tx_r){
      // transmit buffer is empty.
      return;
    }
    
    // can't load the new character in this case.
    if((ReadReg(LSR) & LSR_TX_IDLE) == 0){
      // the UART transmit holding register is full,
      // so we cannot give it another byte.
      // it will interrupt when it's ready for a new byte.
      return;
    }
    
    int c = uart_tx_buf[uart_tx_r];
    uart_tx_r = (uart_tx_r + 1) % UART_TX_BUF_SIZE;
    
    // maybe uartputc() is waiting for space in the buffer.
    wakeup(&uart_tx_r);
    
    WriteReg(THR, c);
  }
}
```
## 部分流程梳理：
当 `$ ` 被发送的时候，相关的流程大致如下所示：自然我们可以直接跑到 consolewrite 这边跑我们的程序，然后呢在 THR 寄存器的操作中，将我们需要传输的字符写入到这个寄存器中。

之后便是硬件上面的事情，硬件通过 BAUD Rate ，由于先前设置了 Transmit bit ，即在 THR 中有值的时候会自动传输，所以其实这个时候只要写了 THR 寄存器，这个信息就会被发送出去。

 UART 保证在发送这个信息的同时，触发一个中断。由于字符是一个一个传输的，` ` 字符也会走完前面 `$` 的完整流程，不过它大概会卡在这个步骤：

感觉会有两种可能性，一种是中断触发的时候下一次 consolewrite 已经 acquire 了 uart_tx_lock ，uartintr 中的 uartstart 却没有看到更新 uart_tx_w ，那么这个中断什么也没做就会返回，之后 ` ` 的传输过程就与 `$` 完全一致。
```C
// add a character to the output buffer and tell the
// UART to start sending if it isn't already.
// blocks if the output buffer is full.
// because it may block, it can't be called
// from interrupts; it's only suitable for use
// by write().
void
uartputc(int c)
{
  acquire(&uart_tx_lock);

  if(panicked){
    for(;;)
      ;
  }

  while(1){
    if(((uart_tx_w + 1) % UART_TX_BUF_SIZE) == uart_tx_r){
      // buffer is full.
      // wait for uartstart() to open up space in the buffer.
      sleep(&uart_tx_r, &uart_tx_lock);
    } else {
      uart_tx_buf[uart_tx_w] = c;
      uart_tx_w = (uart_tx_w + 1) % UART_TX_BUF_SIZE;
      uartstart();
      release(&uart_tx_lock);
      return;
    }
  }
}
```
我尝试用 gdb 跟了一下，看到的结果其实是交替，也就是其实我们依赖 uartputc 来更新 uart_tx_buf ， uartputc 函数中的 uartstart 也会被调用，并不是如同我们在 book 中所看到的那样：typically the first byte will be sent by uartputc’s call to uartstart, and the remaining buffered bytes will be sent by uartstart calls from uartintr as transmit complete interrupts arrive.

不知道未来的版本是否对此有所勘误，不过无所谓，现在对于一开始的 input 流程我已经非常熟练了。


## 其他
出于篇幅，我们不再对上面的代码做整理和解释了，如果想要把脉络做理解，其实也不难：
1. 思考 shell 一开始打印的 $ 字符所生成的全流程 => 不涉及中断的触发，只是使用了 consolewrite 函数打印相应字符到界面上 (其实还是有一些触发？但是没有对 buffer 进行操作)
2. 思考我们输入 ls 之后的全流程 => 中断的触发
3. timerintr 这部分在前面的 labs 中我做了一些简单的代码解读，我们现在就不再梳理了，做完 xv6-labs 之后，我会系统地再梳理完全部代码
