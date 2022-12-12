## Lec2
在这边教材提供的例子是：
```shell
gcc -o add add.c addf.c
```
addf.c中定义了add.c中使用的函数信息，其通过objdump后得到的结果为：
```shell
addf.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <add>:
   0:	8d 04 37             	lea    (%rdi,%rsi,1),%eax
   3:	c3                   	retq  
```
该信息中实际上add函数可以以这样的方式存储：
```C
const unsigned char add[] = { 0x8d, 0x04, 0x37, 0xc3 };
```
下面进行两个尝试：

### 尝试一，直接利用字符串来执行相关的函数操作。

该尝试在笔者环境下失败，与人讨论被告知MAC环境无误。

利用peda调试时得到下列结果：
```shell
[-------------------------------------code-------------------------------------]
   0x40404a:	add    BYTE PTR [rax],al
   0x40404c:	add    BYTE PTR [rax],al
   0x40404e:	add    BYTE PTR [rax],al
=> 0x404050 <add>:	lea    eax,[rdi+rsi*1]
   0x404053 <add+3>:	ret    
   0x404054:	add    BYTE PTR [rax],al
   0x404056:	add    BYTE PTR [rax],al
   0x404058:	add    BYTE PTR [rax],al
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffde18 --> 0x4011a6 (<main+54>:	mov    edi,0x402026)
0008| 0x7fffffffde20 --> 0x401170 (<main>:	push   r14)
0016| 0x7fffffffde28 --> 0x401520 (<__libc_csu_init>:	endbr64)
0024| 0x7fffffffde30 --> 0x0 
0032| 0x7fffffffde38 --> 0x7ffff7de6083 (<__libc_start_main+243>:	mov    edi,eax)
0040| 0x7fffffffde40 --> 0x7ffff7ffc620 --> 0x50f3c00000000 
0048| 0x7fffffffde48 --> 0x7fffffffdf28 --> 0x7fffffffe295 ("/home/link/Desktop/cs300-lectures/datarep/add")
0056| 0x7fffffffde50 --> 0x300000000 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
```
可见我们的指令确实传输成功了，但可能受win环境下的影响依旧检测除了这个问题，能够通过静态检测，但没有通过动态检测。出现了可能的越界问题，但是鄙人没有找到越界突破口。

### 尝试二，利用mmap函数将外界文件读入程序中，利用外界文件的Hex code来执行。

尝试成功。

相关代码：非常简单。
```C
int main(int argc, char* argv[]) {
    if (argc <= 4) {
        fprintf(stderr, "Usage: addin FILE OFFSET A B\n\
    Prints A + B.\n");
        exit(1);
    }

    const char* file = argv[1];
    size_t offset = strtoul(argv[2], 0, 0);
    int a = strtol(argv[3], 0, 0);
    int b = strtol(argv[4], 0, 0);

    // Load the file into memory
    int fd = open(file, O_RDONLY);
    if (fd < 0) {
        die(file);
    }

    off_t size = filesize(fd); // 得到文件的实际长度
    if (size < 0) {
        die(file);
    }

    void* data = mmap(NULL, size, PROT_READ | PROT_EXEC, MAP_SHARED, fd, 0); // 利用mmap将该文件的具体内容读入内存中，以data来存储。
    if (data == MAP_FAILED) {
        die(file);
    }
    uintptr_t data_address = (uintptr_t) data;

    // Call `add`!
    int (*add)(int, int) = (int (*)(int, int)) (data_address + offset);

    printf("%d + %d = %d\n", a, b, add(a, b));
}
```