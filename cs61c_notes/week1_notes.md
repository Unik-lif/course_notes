### Shortcuts:
some useful commands to save time:
As a CLI refresher, when typing commands or file paths:
```
<tab> will autocomplete the current term
<up arrow> and <down arrow> will allow you to refill commands you’ve used previously without typing them again.
<ctrl> + a will move the cursor to the beginning of the current line (helpful for fixing mistakes)
<ctrl> + e will move the cursor to the end of the current line (helpful for fixing mistakes)
<ctrl> + r will let you search through your recently used commands
```
### Man usage:
```
man command_name | less: using less to avoid lengthy comments.
man -k single_keyword | less: search the man page for a command with keyword single_keyword and get a list of all related commands on your system.
```
### Others:
```
scp: Secure Copy. Can copy files between computers using the SSH protocol.
```

Using redirection sends the contents of the input file to stdin.
```
./a.out < filename.txt
```

### Interesting things:
heisenbug: 看样子有点量子力学的味道。观察者效应。

```C
#include <stdio.h>
int main() {
    int a[5] = {1, 2, 3, 4, 5};
    unsigned total = 0;
    for (int j = 0; j < sizeof(a); j++) {
        total += a[j];
    }
    printf("sum of array is %d\n", total);
}
```
根据valgrind的观察，可能出问题的地方在于sizeof(a)这个函数结构，值得进一步研究。

从秦老师那边听来的参看：
出于编译优化的因素，可能造成printf不会出现在有害代码之前运行的情况，具体解决手段有：
1. 从汇编的层面看问题。
2. 利用gdb3的某个特殊选项，将编译优化关闭，便于顺序执行调试。

pay attention to the Makefile usage in the Proj1, maybe someday later we will use it frequently.