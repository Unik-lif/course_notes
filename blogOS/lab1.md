## Preface:
We've finished rCore beforehand, however during my coding process, some details are not taken into consideration. So maybe it make sense for us to use BlogOS as a reference.
### A Freestanding Rust Binary
#### from 'std' to 'no_std'
`Bare metal`: a computer executing instructions directly on logic hardware without an intervening operating system.

In this case, we can't use most of the Rust standard library. But there still are a lot of Rust features that we can use.

`panic_handler`: attribute which defines the function that the compiler should invoke when a panic occurs.

`panic_handler` is provided by standard library, but for [no_std] environment, we need to define it ourselves.

> By default, Rust uses unwinding to run the destructors of all live stack variables in case of a panic. This ensures that all used memory is freed and allows the parent thread to catch the panic and continue execution.

So `eh_personality` item is used for unwinding the stack to get the panic info on the stack.

To disable this functionality, we should add these lines in Cargo.toml.
```
# we abort the unwinding by aborting on panic.
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```
#### start attribute
in a typcial Rust binary that using standard library, execution starts in a C runtime library called `crt0`, which sets up the environment for a C application. This includes creating a stack and placing the arguments in the right registers. The C runtime then invokes the entry point of the Rust runtime -> which is marked by the `start` language item.

but in real time environment 'no-std', we need to define our own entry point.

To tell the Rust compiler that we don’t want to use the normal entry point chain, we add the #![no_main] attribute.
#### Linker errors:
```shell
linkvm@link:/mnt/d/blogos/blog_os$ cargo run
   Compiling blog_os v0.1.0 (/mnt/d/blogos/blog_os)
error: linking with `cc` failed: exit status: 1
```
We should find a way to inform the linker that it should not include the C runtime, we can do this by building for a bare metal target.

Note the `target triple` of our rust is shown below:
```shell
linkvm@link:/mnt/d/blogos/blog_os$ rustc --version --verbose
rustc 1.64.0-nightly (f6f9d5e73 2022-08-04)
binary: rustc
commit-hash: f6f9d5e73d5524b6281c10a5c89b7db35c330634
commit-date: 2022-08-04
host: x86_64-unknown-linux-gnu
release: 1.64.0-nightly
LLVM version: 14.0.6
linkvm@link:/mnt/d/blogos/blog_os$ 
```
Here we can see that the `triple` assumes that there is an underlying operating system such as Linux or Windows that uses the C runtime by default. So, to avoid the linker errors, we can compile for a different environment with no underlying operating system.

If we choose special triple for our target, we can bypass the linker errors here.

in Linux, we can aid the linker error by passing more parameters when compiling.

To solve this, we can tell the linker that it should not link the C startup routine by passing the `-nostartfiles` flag.

One way to pass linker attributes via cargo is the `cargo rustc` command. The command behaves exactly like `cargo build`, but allows to pass options to rustc, the underlying Rust compiler. rustc has the `-C link-arg` flag, which passes an argument to the linker. Combined, our new build command looks like this:

pass the `-nostartfiles` flag to link-arg.
```shell
cargo rustc -- -C link-arg=-nostartfiles
```
To avoid passing parameters like above, we can modify the `.cargo/config.toml` for us to quickly passing our parameters.
```toml
[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]
```
check this document when we need to modify `.cargo/config.toml` 

https://doc.rust-lang.org/cargo/reference/config.html

https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#platform-specific-dependencies

https://doc.rust-lang.org/reference/conditional-compilation.html