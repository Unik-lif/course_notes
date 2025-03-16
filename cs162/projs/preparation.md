## PintOS
参考北大的pintos版本来构建

### 准备工作
提到了一个看起来很有用的Doxygen工具

之后的相关工具在做实验的时候，再具体去落实。

### 编译
确实能够启动，但是有一些困惑
- 为什么这些命令可以启动起来pintos
- 是否真如文档所说的方式来启动pintos
- pintos的Makefile究竟是怎么组织的，为什么最终他们会在build中出现我们想要跑的一系列文件

在utils文件夹中，存在相关命令，如pintos等命令的具体内容细节。这似乎是一种叫做perl的特殊脚本，已经有较长的历史，目前可能用这种脚本的情况比较少，但是它确实能够很方便地面向脚本解析的工作。

编译并且启动pintos的命令具体如下所示
```shell
cd pintos/src/threads/
make
cd build
pintos --
```
Makefile的具体流程是：
- 通过include加载外头的Makefile.kernel内容
- 加载读入Make.vars中设置的变量名信息
- 通过前缀方式把需要的文件包装到build里头，并且把需要的Makefile.build复制进去，对于build下的各个子目录，均跑一下对应


### 细节
- $@表示目标文件，$<表示第一个依赖项
