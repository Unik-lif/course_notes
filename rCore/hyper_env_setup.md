## 简述Hypercraft配置
助教老师[@KuangjuX](https://github.com/KuangjuX)已经在群里说的非常明白了，不过考虑到后入群的同学可能和我一样没法看到之前的信息，导致配置环境可能会多花一些些时间，因此我们还是简单地描述一下这个配环境的过程吧。

首先，我们需要把相关的文件打包下载下来。特别的，由于现在的`HyperCraft`已经作为一个`app`塞到了`ArceOS`之中，我们直接把子模块一起下载下来。
```
git clone --recursive https://github.com/KuangjuX/arceos.git
```
这样我们就能把引用的子模块一并下载过来。
```shell
linkvm@link:/mnt/d/hy$ git clone --recursive https://github.com/KuangjuX/arceos.git
Cloning into 'arceos'...
remote: Enumerating objects: 6241, done.
remote: Counting objects: 100% (2710/2710), done.
remote: Compressing objects: 100% (622/622), done.
remote: Total 6241 (delta 2255), reused 2157 (delta 2081), pack-reused 3531
Receiving objects: 100% (6241/6241), 3.46 MiB | 5.82 MiB/s, done.
Resolving deltas: 100% (3480/3480), done.
Updating files: 100% (464/464), done.
Submodule 'crates/hypercraft' (https://github.com/KuangjuX/hypercraft.git) registered for path 'crates/hypercraft'
Cloning into '/mnt/d/hy/arceos/crates/hypercraft'...
remote: Enumerating objects: 994, done.
remote: Counting objects: 100% (415/415), done.
remote: Compressing objects: 100% (247/247), done.
remote: Total 994 (delta 222), reused 320 (delta 137), pack-reused 579
Receiving objects: 100% (994/994), 10.54 MiB | 9.95 MiB/s, done.
Resolving deltas: 100% (479/479), done.
Submodule path 'crates/hypercraft': checked out '9a2e30b25e952d48e3f4fb47883ca7635c31d555'
linkvm@link:/mnt/d/hy$
```
完成这一步下载之后，我们直接运行助教老师所提供的运行命令：这样我们就会继续下载对应的模块，并用该命令启动，参数的声明请参考`ArceOS`文档。
```shell
make A=apps/hv ARCH=riscv64 LOG=info HV=y run
```
运行这一步的时候你会发现少了一个`img`文件，这个文件已经在仓库中由助教所给出了：[去下载就好](https://pan.baidu.com/s/1WlBcw24raULlj5GPA5Qshw?pwd=jkkz)
```
    Running qemu-system-riscv64 -m 3G -smp 1 -machine virt -bios default -kernel apps/hv/hv_qemu-virt-riscv.bin -device loader,file=apps/hv/guest/linux/linux.dtb,addr=0x90000000,force-raw=on -device loader,file=apps/hv/guest/linux/linux.bin,addr=0x90200000,force-raw=on -drive file=apps/hv/guest/linux/rootfs.img,format=raw,id=hd0 -device virtio-blk-device,drive=hd0 -append root=/dev/vdaqemu-system-riscv64: -drive file=apps/hv/guest/linux/rootfs.img,format=raw,id=hd0: Could not open 'apps/hv/guest/linux/rootfs.img': No such file or directory
make: *** [Makefile:91: justrun] Error 1
```
我们看到对应的位置在`apps/hv/guest/linux/rootfs.img`之中，把文件拷贝过来即可。
```shell
warning: `arceos-hv` (bin "arceos-hv") generated 3 warnings (run `cargo fix --bin "arceos-hv"` to apply 1 suggestion)
    Finished release [optimized] target(s) in 23.37s
rust-objcopy --binary-architecture=riscv64 apps/hv/hv_qemu-virt-riscv.elf --strip-all -O binary apps/hv/hv_qemu-virt-riscv.bin
    Running qemu-system-riscv64 -m 3G -smp 1 -machine virt -bios default -kernel apps/hv/hv_qemu-virt-riscv.bin -device loader,file=apps/hv/guest/linux/linux.dtb,addr=0x90000000,force-raw=on -device loader,file=apps/hv/guest/linux/linux.bin,addr=0x90200000,force-raw=on -drive file=apps/hv/guest/linux/rootfs.img,format=raw,id=hd0 -device virtio-blk-device,drive=hd0 -append root=/dev/vdaqemu-system-riscv64: -device loader,file=apps/hv/guest/linux/linux.bin,addr=0x90200000,force-raw=on: Cannot load specified image apps/hv/guest/linux/linux.bin
make: *** [Makefile:91: justrun] Error 1
```
再次运行，结果发现缺少了一个文件`linux.bin`，这个文件的位置其实是在[这个仓库](https://github.com/KuangjuX/hypercraft/tree/main/guest/linux)中。我们把整个仓库下载下来，利用`git checkout main`切换到`main`分支，将这个`linux.bin`文件拷贝到对应的位置`apps/hv/guest/linux/linux.bin`即可。

再次运行，如果发现`qemu`的位置有误，修改`Makefile`文件中的`QEMU`的`build`地址即可。比如我的`qemu`地址在`QEMUPATH        ?= /mnt/d/qemu-7.0.0/build/`这个位置上。
```shell
linkvm@link:/mnt/d/hy/arceos/crates/hypercraft$ cat Makefile
TARGET          := riscv64gc-unknown-none-elf
MODE            := release

ARCH            ?= riscv64

APP                     ?= hv
APP_ELF         := target/$(TARGET)/$(MODE)/$(APP)
APP_BIN         := target/$(TARGET)/$(MODE)/$(APP).bin
CPUS            ?= 1
LOG                     ?= debug

OBJDUMP     := rust-objdump --arch-name=riscv64
OBJCOPY     := rust-objcopy --binary-architecture=riscv64

GDB                     := riscv64-unknown-elf-gdb

QEMUPATH        ?= your/qemu/build/
QEMU            := $(QEMUPATH)qemu-system-riscv64
BOOTLOADER      := bootloader/rustsbi-qemu.bin
```
修改到这一步就可以运行了：
```
~ #
~ # link
[ 14.069587 0 hypercraft::arch::vm:87] Remote fence is not supported
[   13.972805] __sbi_rfence_v02_call: hbase = [0] hmask = [0x1] failed (error [-524])
[ 14.070273 0 hypercraft::arch::vm:87] Remote fence is not supported
[   13.973537] __sbi_rfence_v02_call: hbase = [0] hmask = [0x1] failed (error [-524])
[ 14.079591 0 hypercraft::arch::vm:87] Remote fence is not supported
[   13.982773] __sbi_rfence_v02_call: hbase = [0] hmask = [0x1] failed (error [-524])
BusyBox v1.37.0.git (2023-03-22 00:31:25 UTC) multi-call binary.

Usage: link FILE LINK

Create hard LINK to FILE
~ #
```