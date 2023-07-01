## Testing
Aim: Use Rust's support for custom test frameworks to execute test functions inside our kernel.

To report the results out of QEMU, we will use different features of QEMU and the bootimage tool.

Rust has a built-in test framework, which is capable of running unit tests without the need to set anything up.

But this depends on the built-in `test` library, which depends on the standard library. This means that we can't use the default test framework for our `#[no_std]` kernel.

Please follow the steps in the Post.

key words: custom_test_frameworks
### How can we exit the QEMU
Luckily, there is an escape hatch: QEMU supports a special isa-debug-exit device, which provides an easy way to exit QEMU from the guest system. To enable it, we need to pass a -device argument to QEMU. We can do so by adding a package.metadata.bootimage.test-args configuration key in our Cargo.toml:
### Printing to the Console
So far, our qemu window will quickly close, maybe we should find a way to keep our results in the qemu window.

The use of serial port and the use of VGA buffer is quite different.

```
"-serial", "stdio"
```
direct serial output to stdio.

When we set `test-timeout`, if we cause a time out, the result is shown below:
```shell
Error: Test timed out
error: test failed, to rerun pass '--bin blog_os'
make: *** [Makefile:12: test] Error 1
```
### integration tests:
In Rust, 'integration tests' means to put tests into a tests directory in the project root.

integration will be useful when our kernel becomes larger and more complex.