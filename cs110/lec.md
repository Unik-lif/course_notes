## Lec 01
RWX: -> umask. What permission you can set after masking.

0: 八进制

errno -l：Unix中遇到的问题

## Lec 02
Lambda function in C++:

```
[captures] (params) { body }
```

#### advantage of lambda function:
- solve problems that can't be dealed with function pointer.

Hard for function pointer to use parameter that we don't want to pass, like local variables.

I've solved this in no concise ... The key is to using `[captures]`

If you want to capture al class variables, you can use `[this]` as a capture clause.

#### Another program
tee.

`stat` and `lstat`: the first returns info about the file the link references, and the second returns info about the link itself.

