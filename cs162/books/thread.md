## 阅读记录
### Chapter 4
Threads是一种危险的抽象，对其使用需要有一些特殊的理由，而好处也是显而易见
- expressing logically concurrent tasks
- shifting work to run in the background: 关闭键
- exploiting multiple processors
- managin I/O devices with seperate thread

Thread Control Block
- 存放thread运行的计算状态
- 存放用于管理线程的Metadata

Shared States
- 虽然thread对于per-thread state是由存储，但是某些变量允许threads之间进行共享
- program code，global variables，以及heap中的数据是可以被共享的
- 但是线程自己也有thread-local类型的变量，比如errno，表示线程自己对于最近的系统调用的错误返回值

危险之处：本质上还是共享地址空间，逻辑上分割开来，但是现实没有分割开来，存在被篡改的风险


