## Set up lab
### VM vs Container
cited from the url below: [Post.](https://www.backblaze.com/blog/vm-vs-containers/)

Virtualization: create multiple instances from the hardware and software in a single machine.

Hypervisor: a lightweight software layer that sits between the physical hardware and the virtualized environments and allows multiple operating systems to run in tandem on the same hardware. 

THE key difference is the size. VM not only a full copy of OS, but also a virtual copy of the hardware. So VM can be very monolithic. But Containers simply virtualize the OS.

The VM should be isolated with each other, that's the basic function of HYPERVISOR.

As for container, each of them shares the host OS kernel and, usually, the binaries and libraries, which significantly reduces the need to reproduce the operating system code.

Besides, because of its light-weight, the programmer will therefore get rid of tedious bugs in OS, hardware and many other complex stuffs.

(see post mentioned above to peek at the models.)

