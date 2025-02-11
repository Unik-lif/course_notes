## 实验记录
做一个阉割版的mmap。

```C
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```
其中addr一直都是0，mmap来决定地址的起始位置，要么是一个正确的值，要么是0xfffffff表示失败。其中prot表示这个内存应该被映射的情况，而flag则比较特殊，表示MAP_SHARED的时候，最后的更改需要落实到文件中，而如果是MAP_PRIVATE，则说明不能落实到文件里。

好像前置条件还挺简单的，之后就跟着HINTS一步一步做着来。
