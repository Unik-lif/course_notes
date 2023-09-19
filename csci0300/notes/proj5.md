## Vunmo
1. 我感觉对于`users`较少的情况，可能对于每个请求分别进行`spawn`是更有经济效益和效率的一种做法。但用户多起来了，建立`worker pool`可能效果更好一些？
2. 既然是同时进行的话，那么两边各自完成第一行后就全部都阻塞住了，没法继续往下运行了。

对于条件变量的判别需要用`while`来进行包装。有一种现象被称为`spurious wakeup`，即便在条件没有满足的情况下也有唤醒线程的可能性。

完成了第一部分同步队列的撰写。

### Server Design:
实现服务器的功能主要还是要满足下面这三个功能：
1. 监听并且接受来自用户的连接请求。
2. 接收并且读取用户的连接请求。
3. 需要有几个用于处理请求的工作线程，并且能够向用户响应。

### 一些C++的细节知识点：
1. 对于`non-static`类型的函数，其函数不会自动转化为地址（即函数指针），需要用`&`来转义。
2. 编译器在调用`non-static`类型函数于`thread`中时，需要确定作用的实例，即对象，所以需要加上`this`关键字。
3. 对线程调试时要思考清楚当前错误的模型，分析清楚原因后再去尝试。
4. 这边的线程`gdb`调试很有收获。
### 很好，完成了：
```
 === queue ===
1. [PASSED] serial_push_all_flush
2. [PASSED] serial_push_all_pop_all
3. [PASSED] serial_push_pop
4. [PASSED] threads_push_all_pop_all
5. [PASSED] threads_push_pop

 === server_part1 ===
6. [PASSED] test1_start_stop
7. [PASSED] test2_accept_clients
8. [PASSED] test3_process_requests

 === server_part2 ===
9. [PASSED] basic-load-test
10. [PASSED] charge-request-declined
11. [PASSED] charge-request-insufficient-balance
12. [PASSED] charge-request-invalid-target
13. [PASSED] charge-request-success
14. [PASSED] client-setup
15. [PASSED] deposit-request
16. [PASSED] pay-request-insufficient-balance
17. [PASSED] pay-request-invalid-target
18. [PASSED] pay-request-success
19. [PASSED] withdraw-request

=======================
Tests Passed: 19 / 19
=======================
```