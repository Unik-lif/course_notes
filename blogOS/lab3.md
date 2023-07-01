## VGA Text Mode
a simple way to print text to the screen.

VGA text buffer is a two-dimensional array with typically 25 rows and 80 columns, which is directly rendered to the screen.

VGA: first byte: ascii byte, second byte: color byte.

reads and writes to the address of 0xb8000 don't access the RAM but directly access the text buffer on the VGA hardware. This means we can read and write it through normal memory operations to that address.

We've conducted this part in memory already, but can we find a better way to use this without using unsafe block in rust?

many interesting functions we have been implemented here! Full of fun in fact!

### lazy_static.
`statics` in rust are initialized at compile time, in contrast to normal variables that are initialized at run time.

Rust use const evaluator to evaluate the variables we called. The one-time initialization of statics with non-const functions is a common problem in Rust.

`lazy_static!` macro defines a lazily initialized `static`. Instead of computing its value at compile time, the `static` lazily initialized `static`, instead of computing its value at compile time, the `static` lazily initializes itself when accessed for the first time.
### interior mutability
it is numb to have a `Writer` that can't be changed.

We could try to use an immutable static with a cell type like RefCell or even UnsafeCell that provides interior mutability. But these types aren’t Sync (with good reason), so we can’t use them in statics.

-> spinlocks, but it burns CPU until the mutex is free again.

> Thanks to cargo, we also saw how easy it is to add dependencies on third-party libraries. The two dependencies that we added, lazy_static and spin, are very useful in OS development and we will use them in more places in future posts.