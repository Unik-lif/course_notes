## Testing
Aim: Use Rust's support for custom test frameworks to execute test functions inside our kernel.

To report the results out of QEMU, we will use different features of QEMU and the bootimage tool.

Rust has a built-in test framework, which is capable of running unit tests without the need to set anything up.

But this depends on the built-in `test` library, which depends on the standard library. This means that we can't use the default test framework for our `#[no_std]` kernel.

Please follow the steps in the Post.

key words: custom_test_frameworks
### How can we exit the QEMU
Luckily, there is an escape hatch: QEMU supports a special isa-debug-exit device, which provides an easy way to exit QEMU from the guest system. To enable it, we need to pass a -device argument to QEMU. We can do so by adding a package.metadata.bootimage.test-args configuration key in our Cargo.toml:

