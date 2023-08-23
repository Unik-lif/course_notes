## Guest Module For VMPL-SGX.
To better enable the ability of the SVSM-SGX, We set up a guest module here.

To fulfill the need of user apps to use new features added by Guset OS and SVSM, besides the enclave the SVSM provides, it is irresistable to add a new module.

### What's the magic?
The role this module plays is to create an interface for apps in vmpl1 & ring 3 to use syscall to act on enclaves.

Draw a simple picture here is necessary.

|      \      |     ring 0  |    ring 3     |
| :----:      |    :----:   |    :----:     |
|  vmpl 1     |  Guest OS   |  Apps         |
|  vmpl 0     |   SEH       |  Enclave      |
### Design goals:
1. provide basic interface for user apps.
2. call `vmmcall` in a certain way to let SEH behave in a proper way. -> How to Seperate its behavior from others?

### How can we add new features in Guest OS?
1. Use the specific Kernel version for Guest OS to compile.
2. Create a device for Apps to interface.

### Control Flow Details.
1. The user App first **open** the device.
2. Then it use **ioctl** to conduct some operations. In response, the device will convey the cmd arguments for the user apps. These parameters will be sent to the Hypervisor in VMSA data structure, and Hypervisor will transfer vmpl1 to vmpl0.
3. The Hypervisor responses and send parameters wrapped by VMSA back to the svsm.bin. SVSM can handle the VMSA and use protocols to update the current state in vmpl0.

### Questions remain:
- How can the Guest OS Device seperate itself from other common `VMGEXIT`? Please Read the Source Code and Answer this question.
### What's the role SEH will play?
- catch the execution flow of Guest OS to communicate with the Hypervisor.
- That is, when the guest needs to execute something at VMPL0 or to request other services from the SVSM, it follows the SVSM API and requests the hypervisor to run the VMPL0 VMSA that is associated with the SVSM.
- Guest OS module will first trap to HV, and then shift to VMPL0, where the SEH situates.

### Usage:
Please refer to Makefile.

For very strange reasons, this module can't be successfully been built in Guest OS. So A releative handy way might be making the module outside the guest OS, then, we can simply type the following commands:
```
sudo scp -r link@192.168.112.29:/home/link/linux-svsm/guest guest
```
Therefore the whole directory will be passed into the VM, we can use the following commands to repair the Guest OS.
```
make clean
make && make modules_install && make install
```
### Tips:
The Kernel version the Guest OS use is `linux-image-6.2.0-snp-guest-20e69de4dd39+`, that is a little different from the Kernel that built by `<guest.deb>` in linux-svsm.

It is recommanded that build guest os step by step. The package gain in the Host Machine can sometimes by broken, and therefore can't build new modules that we want.

### Author:
Link Song: link23er@gmail.com

MIT Licenses.