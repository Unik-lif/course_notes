## A Minimal Rust Kernel
when we turn on a computer, it begins executing firmware code that is stored in motherboard `ROM`. 

This code performs a power-on self-test, detects available RAM and pre-initializes the CPU and hardware.

Then, it looks for a bootable disk and starts booting the operating system kernel.

Two kinds of firmware standards on x86:
1. BIOS.
2. UEFI.

Before the BIOS Boots, the CPU is put into a 16-bit compatibility mode called `real mode`. -> JOS has shown this.

16-bit real mode -> 32-bit protected mode -> 64-bit long mode.

Use a tool nameed `bootimage` to help our work.

Multiboot -> GNU GRUB (standard)
```json
    "linker-flavor": "ld.lld",
    "linker": "rust-lld",
    "panic-strategy": "abort",
    "disable-redzone": true,
    "features": "-mmx,-sse,+soft-float",
```
`red-zone`: an optimization that may trigger faults when interrupts happen, so basically we should close this ability.

"features" -> disable SSE, MMX and some SIMD instructions. soft-float -> float ability.

for custom triples like what involved in blog_os, `core` crate in Rust might be missing. That's why we should choose `nightly` version of Rust, which can aid this obstacle using "build-std".

Using this, we can reinstall some crates like core.

What's more, since We've used no-mangle already, so we can enable build-std-features. This will help us to use Some basic libc with colliding with the C library.

We can use `bootimage` to link the os with our bootloader. Simply following the steps in the post, and we can see `bootimage-blog_os.bin` afterwards.