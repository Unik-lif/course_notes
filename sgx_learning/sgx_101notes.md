简单过一遍`galtech`提供的`Intel SGX`的学习材料。
# Overview章节
大体上这一章节我还是看得比较清楚的，不过应该还是有必要稍微看一点其他的扩展材料。

## 材料一：http://www.cs.tau.ac.il/~tromer/istvr1516-files/lecture10-trusted-platform-sgx.pdf
### 今世前生
似乎确实和可信启动有点关系，在启动后似乎使用的是`PCR values`的值来实现安全上的度量工作。

也有一些远程认证的一些需求，通过`TPM`来向遥远的组织告知当前在我的机器上运行的是什么软件。
```
Good applications:
 Bank allows money transfer only if customer’s 
machine runs “up-to-date” OS patches.
 Enterprise allows laptop to connect to its network only 
if laptop runs “authorized” software
 Quake players can join a Quake network only if their 
Quake client is unmodified.
```
### 需求：
`Intel SGX`是为了克服`TXT`会停止`OS`的困难，以及信任根过大以及开发难度过大这样的问题。并且，`SGX`能够支持长期的`trusted`和`untrusted`应用程序的运行。

`SGX`首次带来了`Enclave`的概念，其有效减少了`TCB`大小和可能遇到的攻击面。当然，其也存在着还是得由`untrusted OS`来管理资源的缺陷。