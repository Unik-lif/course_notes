## Lab0
主要是为了搞清楚从PC上电之后，到操作系统跑起来之时，中间经历的过程。

### Loader
pintos最先跑起来的部分似乎是loader。首先，机器的BIOS将会把loader放到内存中，之后loader将会找到磁盘中的内核，将其装载进入到内存中，并跳转到其中，使得内核能够跑起来。

其中的 pintos -- 内容似乎是下面的，可以看出他确实经过了比较完备的包装
```
qemu-system-i386 -device isa-debug-exit -drive format=raw,media=disk,index=0,file=/tmp/JuDBbiRziN.dsk -m 4 -net none -nographic -monitor null
```
甚至感觉有点像魔法，真令人惊讶

