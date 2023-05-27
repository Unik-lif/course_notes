## 并发
## 实验：
先尝试考虑把前一部分移植吧，我们需要完成`6b`对应的基本移植工作。

首先我们仅仅把`sys_get_time`移植了，看看效果：
```shell
[PASS] found <sync_sem passed111042126941886!>
[PASS] found <Test write B OK111042126941886!>
[FAIL] not found <deadlock test mutex 1 OK111042126941886!>
[PASS] found <Test power_7 OK111042126941886!>
[PASS] found <forktest pass.>
[PASS] found <pipetest passed111042126941886!>
[PASS] found <race adder using spin mutex test passed111042126941886!>
[PASS] found <file_test passed111042126941886!>
[PASS] found <hello child process!>
[PASS] found <philosopher dining problem with mutex test passed111042126941886!>
[PASS] found <threads with arg test passed111042126941886!>
[PASS] found <exit pass.>
[PASS] found <Hello, world from user mode program!>
[PASS] found <test_condvar passed111042126941886!>
[PASS] found <child process pid = (\d+), exit code = (\d+)>
[PASS] found <Test write C OK111042126941886!>
[PASS] found <threads test passed111042126941886!>
[PASS] found <deadlock test semaphore 2 OK111042126941886!>
[PASS] found <mpsc_sem passed111042126941886!>
[FAIL] not found <deadlock test semaphore 1 OK111042126941886!>
[PASS] found <Test write A OK111042126941886!>
[PASS] found <Test power_5 OK111042126941886!>
[PASS] found <Test power_3 OK111042126941886!>
[PASS] not found <FAIL: T.T>
[PASS] not found <Test sbrk failed!>
```
如果我们把`sys_get_time`去掉，得到的结果是：
```shell
[PASS] found <Test power_3 OK5829231262111042126941886!>
[PASS] found <pipetest passed5829231262111042126941886!>
[PASS] found <test_condvar passed5829231262111042126941886!>
[FAIL] not found <deadlock test mutex 1 OK5829231262111042126941886!>
[PASS] found <race adder using spin mutex test passed5829231262111042126941886!>
[PASS] found <threads with arg test passed5829231262111042126941886!>
[PASS] found <exit pass.>
[PASS] found <Test write C OK5829231262111042126941886!>
[PASS] found <sync_sem passed5829231262111042126941886!>
[PASS] found <philosopher dining problem with mutex test passed5829231262111042126941886!>
[PASS] found <hello child process!>
[PASS] found <threads test passed5829231262111042126941886!>
[PASS] found <forktest pass.>
[FAIL] not found <deadlock test semaphore 1 OK5829231262111042126941886!>
[PASS] found <Test power_7 OK5829231262111042126941886!>
[PASS] found <file_test passed5829231262111042126941886!>
[PASS] found <mpsc_sem passed5829231262111042126941886!>
[PASS] found <Test write B OK5829231262111042126941886!>
[PASS] found <Hello, world from user mode program!>
[PASS] found <Test write A OK5829231262111042126941886!>
[PASS] found <Test power_5 OK5829231262111042126941886!>
[FAIL] not found <deadlock test semaphore 2 OK5829231262111042126941886!>
[PASS] found <child process pid = (\d+), exit code = (\d+)>
[PASS] not found <FAIL: T.T>
[PASS] not found <Test sbrk failed!>
```