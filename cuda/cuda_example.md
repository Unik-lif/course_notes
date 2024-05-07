# Book review
## Chapter 1
画饼的一章，无所谓
## Chapter 2
搭建实验环境。先前已经在所里服务器中装好了英伟达驱动，输入 nvidia-smi 得到下面的结果：
```
Tue May  7 15:12:34 2024
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.40.07              Driver Version: 550.40.07      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce GTX 1080 Ti     Off |   00000000:A1:00.0 Off |                  N/A |
|  0%   41C    P8             19W /  250W |     571MiB /  11264MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A    623811      G   ...sion,SpareRendererForSitePerProcess         70MiB |
|    0   N/A  N/A    729193      G   ...ink/.nutstore/dist/jre/bin/nutstore          2MiB |
|    0   N/A  N/A   1922574      G   /usr/lib/xorg/Xorg                            383MiB |
|    0   N/A  N/A   1922742      G   /usr/bin/gnome-shell                           10MiB |
|    0   N/A  N/A   2413354      G   ...86,262144 --variations-seed-version        100MiB |
+-----------------------------------------------------------------------------------------+
```
跑 CUDA 似乎需要两个不同的编译器
## Chapter 3
host code and device code are different.

Host: CPU 和内存

Device: GPU 和显存

There is nothing special about this; it is shorthand to send host code to one compiler and device code to another compiler. The trick is actually in calling the device code from the host code. One of the benefits of CUDA C is that it provides this language integration so that device function calls look very much like host function calls

对于下面的这一行中<>等标志的解释：
```C
kernel<<<1,1>>>();
```
These are not arguments to the device code but are parameters that will influence how the runtime will launch our device code.

对于 device 中使用的变量，需要特别使用 cudaMalloc 来进行内存分配，之后才能进行使用。

前三章简单地过完了。明天过接下来的三章，争取本周内看完该书。