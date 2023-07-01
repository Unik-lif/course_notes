## 第七章：进程间通信
这章似乎没有具体的任务，我们看代码就够了。

总体而言，第七章的框架代码似乎并没有太大的变化，仅仅是增加了一些系统调用的接口。
### 管道：
`pipe`为文件提供了读写的端口，当然，这个功能依托于`PIPE`类型，它在文件系统中是被定义为有`File`特性的存在。

根据`GNU Man Page`上的信息，`pipe_fd`是一个存储管道两端的读写文件描述符的数组。
```C
int pipe(int pipefd[2]);
```
`pipe`函数将会建立一个单向的、进程间通信用的管道，任何写入管道写端的数据直到有读操作执行时，都会在内核所对应的缓冲区中储存，在`pipefd[0]`将会对应读端，`pipefd[1]`将会对应写端。

只有所有管道端口都被关闭掉，管道占用的资源才会回收。管道建立的初衷是单向，所以在子进程`fork`了相应的文件描述符，即管道信息后，父进程和子进程需要各自关闭掉一个，以方便进程间的读写通信。如果想要实现父子进程之间的双向通信，似乎要建立两个管道才比较合适。

在测例的`ch7b_pipe_large_test`中可以找到这个玩意儿。

如何关闭掉端口，即丢弃掉文件描述符，这边的做法是：
```rust
// os/src/syscall/fs.rs

pub fn sys_close(fd: usize) -> isize {
    let task = current_task().unwrap();
    let mut inner = task.acquire_inner_lock();
    if fd >= inner.fd_table.len() {
        return -1;
    }
    if inner.fd_table[fd].is_none() {
        return -1;
    }
    inner.fd_table[fd].take();
    0
}
```
利用`take`把原本存在的东西丢弃掉，以`None`来代替之。

在上面我们提到管道本质上在这边被以文件类型来解释，我们来看一下这件事情是怎么发生的。
```rust
pub struct Pipe {
    readable: bool,
    writable: bool,
    buffer: Arc<UPSafeCell<PipeRingBuffer>>,
}

impl Pipe {
    /// create readable pipe
    /// 不允许向读端写
    pub fn read_end_with_buffer(buffer: Arc<UPSafeCell<PipeRingBuffer>>) -> Self {
        Self {
            readable: true,
            writable: false,
            buffer,
        }
    }
    /// create writable pipe
    /// 不允许向写端读
    pub fn write_end_with_buffer(buffer: Arc<UPSafeCell<PipeRingBuffer>>) -> Self {
        Self {
            readable: false,
            writable: true,
            buffer,
        }
    }
}
```
`Pipe`由可写可读的性质以及自身的`buffer`包装而成，其中我们还能看到针对读端与写端分别有不同的初始化方式，分别对应了数组`pipefd`的`0`号和`1`号描述符的类型。

上述的`buffer`是读写缓冲区，简单看一下这个数据结构是怎么做的：
```rust
const RING_BUFFER_SIZE: usize = 32;

#[derive(Copy, Clone, PartialEq)]
enum RingBufferStatus {
    Full,
    Empty,
    Normal,
}

pub struct PipeRingBuffer {
    // 最大可以容纳32个字节，这边打算用循环数组来做这件事情
    arr: [u8; RING_BUFFER_SIZE],
    // 数据的头在哪里，这边是读的位置
    head: usize,
    // 数据的尾在哪里，这边是写的位置
    tail: usize,
    // 当前的状态，如上所示
    status: RingBufferStatus,
    // 写端口
    // 利用写端口来确定是否所有的写端口都已经被关闭了
    // 这里的写端口指的其实是read和write的两种类型
    // 如果端口开放，此处存放的是强引用，否则将会存放弱引用
    // 弱引用表示并没有占据所有权，这也意味着Arc计数时它不算，如果强引用不存在，则会自动drop掉信息以防止内存泄露
    write_end: Option<Weak<Pipe>>,
}
```
对于这个`PipeRingBuffer`的读写，其实好像没什么好说的，看代码就好了，了解到这是一个循环数组并且读写各有一个小指针就好了。特别的，`read`是把东西读出来，而`write`是把东西写进去。

有两个函数还是稍微看一下：
```rust
    pub fn set_write_end(&mut self, write_end: &Arc<Pipe>) {
        // 设置write_end是一个weak链接
        self.write_end = Some(Arc::downgrade(write_end));
    }
    
    pub fn all_write_ends_closed(&self) -> bool {
        // 看write_end是否还存在，如果是is_none，说明升格而来的东西已经无了
        // write_end的存在意义，即去查看我的Buffer是否当前是有所有者的？
        // 即，是否有端口类型的文件描述符持有我？这是一个有意义的问题
        self.write_end.as_ref().unwrap().upgrade().is_none()
    }
```
我们暂且不关心这个读写怎么进行的，先看看创建管道时发生了什么：
```rust
/// Return (read_end, write_end)
pub fn make_pipe() -> (Arc<Pipe>, Arc<Pipe>) {
    let buffer = Arc::new(unsafe { UPSafeCell::new(PipeRingBuffer::new()) });
    let read_end = Arc::new(Pipe::read_end_with_buffer(buffer.clone()));
    let write_end = Arc::new(Pipe::write_end_with_buffer(buffer.clone()));
    // buffer 本身并没有对于Arc的所有权，它被降级了，真正的Arc所有权掌握在read_end和write_end之上
    // buffer的意义在于他是一个观测者，检查是否有文件描述符当前拥有自己
    buffer.exclusive_access().set_write_end(&write_end);
    (read_end, write_end)
}
```
非常简单不是吗，创建一个内存，然后克隆给后面的一个读端口一个写端口。对应的系统调用如下所示：
```rust
pub fn sys_pipe(pipe: *mut usize) -> isize {
	trace!("kernel:pid[{}] sys_pipe", current_task().unwrap().pid.0);
    let task = current_task().unwrap();
    let token = current_user_token();
    let mut inner = task.inner_exclusive_access();
    let (pipe_read, pipe_write) = make_pipe();
    // 寻找文件描述符与读写端的管道所对应起来
    let read_fd = inner.alloc_fd();
    inner.fd_table[read_fd] = Some(pipe_read);
    let write_fd = inner.alloc_fd();
    inner.fd_table[write_fd] = Some(pipe_write);
    // pipe表示应用地址空间的数组的起始地址，除了需要在TCB对应的内核地址空间中写fd_table以外，
    // 需要文件描述符写回
    *translated_refmut(token, pipe) = read_fd;
    *translated_refmut(token, unsafe { pipe.add(1) }) = write_fd;
    0
}
```
为了能把`pipe`对应的文件描述符更好地使用起来，针对`File`特征的相关函数，程序做了下面的设置：
```rust
impl File for Pipe {
    fn readable(&self) -> bool {
        self.readable
    }
    fn writable(&self) -> bool {
        self.writable
    }
    fn read(&self, buf: UserBuffer) -> usize {
        // 首先判断管道是否是读端口
        assert!(self.readable());
        let want_to_read = buf.len();
        let mut buf_iter = buf.into_iter();
        let mut already_read = 0usize;
        // 循环的意思其实有点像忙等，如果管道内没有数据可以读，会一直等其他进程把东西写进来
        loop {
            let mut ring_buffer = self.buffer.exclusive_access();
            // 检查还有多少可以读
            let loop_read = ring_buffer.available_read();
            // 如果是空的
            if loop_read == 0 {
                // 如果这个读管道对应的buffer已经失去了全部的强引用，说明没有文件描述符拥有这个buffer了
                // 说明相关文件描述符已经被关闭了，即已经用sys_close关掉了fd，进而已经被自动drop掉了
                // 自然什么都读不出来
                if ring_buffer.all_write_ends_closed() {
                    return already_read;
                }
                drop(ring_buffer);
                // 让下一个进程乃至之后的进程跑起来（毕竟我们在循环内），等待他们的写入
                suspend_current_and_run_next();
                continue;
            }
            // 如果能读，那么就逐个字节地读取
            // 逐个地址地获取buf内的字节，然后直接写他们就好了
            // 有一说一，这个写法其实是冗余的，因为buf_iter.next()的循环判别和want_to_read其实是同一个东西，
            // 相比原来的写法的好处在于我可以少一些判别？可以更早返回？
            // 其实只要注意到一件事：如果我们还没达到want_to_read
            // 会等到下一个loop有其他文件写进来后继续累加
            for _ in 0..loop_read {
                if let Some(byte_ref) = buf_iter.next() {
                    unsafe {
                        *byte_ref = ring_buffer.read_byte();
                    }
                    already_read += 1;
                    if already_read == want_to_read {
                        return want_to_read;
                    }
                } else {
                    return already_read;
                }
            }
        }
    }
    // 反向操作，没有什么特别不一样的地方
    // already_write的方法论：如果当前时刻写不完，那么就让下一些进程继续写，我们等到达到want_to_write的上限后再写
    // 或者等到buf已经迭代到最后一个位置了再说
    fn write(&self, buf: UserBuffer) -> usize {
        assert!(self.writable());
        let want_to_write = buf.len();
        let mut buf_iter = buf.into_iter();
        let mut already_write = 0usize;
        loop {
            let mut ring_buffer = self.buffer.exclusive_access();
            let loop_write = ring_buffer.available_write();
            if loop_write == 0 {
                drop(ring_buffer);
                suspend_current_and_run_next();
                continue;
            }
            // write at most loop_write bytes
            for _ in 0..loop_write {
                if let Some(byte_ref) = buf_iter.next() {
                    ring_buffer.write_byte(unsafe { *byte_ref });
                    already_write += 1;
                    if already_write == want_to_write {
                        return want_to_write;
                    }
                } else {
                    return already_write;
                }
            }
        }
    }
}
```
### 命令行参数：
为了支持命令行参数，需要在`sys_exec`处添加参数接口。

之前我们的`shell`是不支持命令行参数的，现在我们输入的`line`可能有一些命令行参数，所以要先想办法把`line`进行分割处理。
```rust
let args: Vec<_> = line.as_str().split(' ').collect();
let mut args_copy: Vec<String> = args
.iter()
.map(|&arg| {
    let mut string = String::new();
    string.push_str(arg);
    string
})
.collect();

args_copy
.iter_mut()
.for_each(|string| {
    string.push('\0');
});
```
分割成字符串后，往后面添一个0。
```rust
// user/src/bin/ch6b_user_shell.rs

let mut args_addr: Vec<*const u8> = args_copy
.iter()
.map(|arg| arg.as_ptr())
.collect();
args_addr.push(0 as *const u8);
```
收集字符串的起始地址，放在`args_addr`之中，进来之后，还要通过虚实地址转化，把真正的字符串从相应的地址中读出来。
```rust
    loop {
        let arg_str_ptr = *translated_ref(token, args);
        if arg_str_ptr == 0 {
            break;
        }
        args_vec.push(translated_str(token, arg_str_ptr as *const u8));
        unsafe {
            args = args.add(1);
        }
    }
```
压栈的过程写的太妙了，建议对照图片进行理解，感觉很神！

https://learningos.github.io/rCore-Tutorial-Guide-2023S/chapter7/2cmdargs-and-redirection.html#sys-exec

### 重定向：
一个进程无从知道重定向的事实，这对于他们来说是透明的事情，在应用中除非明确指出了数据要从制定的文件输入或者输出到指定的文件，否则文件的输入输出均是`STDIN`和`STDOUT`。

为了改变应用进程的文件描述符输出方向，需要引入重定向的系统调用：`sys_dup`。

具体来说这个系统调用将会把进程中一个已经打开的文件复制一份并且分配到一个新的文件描述符中，如果出现了错误则返回`-1`，否则访问已经打开文件的新文件描述符。
```rust
pub fn sys_dup(fd: usize) -> isize {
	trace!("kernel:pid[{}] sys_dup", current_task().unwrap().pid.0);
    let task = current_task().unwrap();
    let mut inner = task.inner_exclusive_access();
    if fd >= inner.fd_table.len() {
        return -1;
    }
    if inner.fd_table[fd].is_none() {
        return -1;
    }
    let new_fd = inner.alloc_fd();
    inner.fd_table[new_fd] = Some(Arc::clone(inner.fd_table[fd].as_ref().unwrap()));
    new_fd as isize
}
```
可以直接把他理解成`duplicate`。

之后，需要检查`shell`中是否有重定向的标志符号，这边假设输入的方式是合法的：需要注意的是，不管重定向作为输入还是输出，紧随其后的东西才是真正我们要读写操作的东西。
```rust
// user/src/bin/ch6b_user_shell.rs

// redirect input
let mut input = String::new();
if let Some((idx, _)) = args_copy
.iter()
.enumerate()
.find(|(_, arg)| arg.as_str() == "<\0") {
    input = args_copy[idx + 1].clone();
    args_copy.drain(idx..=idx + 1);
}

// redirect output
let mut output = String::new();
if let Some((idx, _)) = args_copy
.iter()
.enumerate()
.find(|(_, arg)| arg.as_str() == ">\0") {
    output = args_copy[idx + 1].clone();
    args_copy.drain(idx..=idx + 1);
}
```
对于`shell`程序，如果要实现重定向，接下来这段就是典中典了：
```rust
// user/src/bin/user_shell.rs

let pid = fork();
if pid == 0 {
    // input redirection
    if !input.is_empty() {
        // 作为input的重定向者，我手头有他的名字，那太好了，我打开它就好了
        let input_fd = open(input.as_str(), OpenFlags::RDONLY);
        if input_fd == -1 {
            println!("Error when opening file {}", input);
            return -4;
        }
        let input_fd = input_fd as usize;
        // 关闭STDIN，防止造成干扰
        close(0);
        // 这个文件副本分配到的fd是0，从而利用文件代替了原本的STDIN数据？
        assert_eq!(dup(input_fd), 0);
        close(input_fd);
    }
    // output redirection
    if !output.is_empty() {
        let output_fd = open(
            output.as_str(),
            OpenFlags::CREATE | OpenFlags::WRONLY
        );
        if output_fd == -1 {
            println!("Error when opening file {}", output);
            return -4;
        }
        let output_fd = output_fd as usize;
        close(1);
         // 这个文件描述符分配到的fd是1，从而利用文件代替了原本的STDOUT数据？
        assert_eq!(dup(output_fd), 1);
        close(output_fd);
    }
    // child process
    if exec(args_copy[0].as_str(), args_addr.as_slice()) == -1 {
        println!("Error when executing!");
        return -4;
    }
    unreachable!();
} else {
    let mut exit_code: i32 = 0;
    let exit_pid = waitpid(pid as usize, &mut exit_code);
    assert_eq!(pid, exit_pid);
    println!("Shell: Process {} exited with code {}", pid, exit_code);
}
```

到这边，本章的全部代码均解读完毕。