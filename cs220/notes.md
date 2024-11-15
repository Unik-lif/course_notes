## KAIST CS220 Crash
Rust benefits of Modern language:
- productive
- type-safe, functional, parallelism-friendly
- safe pointer programming


## Notes:
- Map is a lazy functional function.

- two kinds of crates: `lib.rs` or `main.rs`, for library form and binary form.

- The mod keyword declares modules, and Rust looks in a file with the same name as the module for the code that goes into that module.

- `or_insert_with` and `as_mut` is a very good tool.

- Unit Tests: test each unit of code in isolation from the rest of the code to quickly pinpoint where code is and isn't working as expected.

- #[cfg(test)]: only run the test code when running cargo test.

- In Rust test, we can testing private functions.

## Progress:
assignment 1/2: done in 2024/09/14
assignment 3: done in 2024/11/02 => A little hard.
assignment 4: done in 2024/11/15 => kinda of challenge!!

```
Running with cargo...

running 3 tests
test assignments::assignment04::grade::test::test_context_calc_expression ... ok
test assignments::assignment04::grade::test::test_context_calc_command ... ok
test assignments::assignment04::grade::test::test_parse ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 104 filtered out; finished in 0.00s

Your score: 1/1
```

