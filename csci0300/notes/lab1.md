## Makefile and gcc
### gcc flag.
The gcc compiler supports the use of hundreds of different flags, which we can use to customize the compilation process. Flags, typically prefixed by a dash or two (-<flag> or --<flag>), can help us in many ways from warning us about programming errors to optimizing our code so that it runs faster.

The general structure for compiling a program with flags is:
```
$ gcc <flags> <c-files> -o <executable-name>
```
Warning Flags:

- **-Wall**: One of the most common flags is the -Wall flag. It will cause the compiler to warn you about technically legal but potentially problematic syntax, including:
- - Uninitialized and unused variables
- - Incorrect return types
- - Invalid type comparisons


- **-Werror**: The -Werror flag forces the compiler to treat all compiler warnings as errors, meaning that your code won’t be compiled until you fix the errors. This may seem annoying at first, but in the long run, it can save you lots of time by forcing you to take a look at potentially problematic code.
- **-Wextra**: 
This flag adds a few more warnings (which will appear as errors thanks to -Werror, but are not covered by -Wall. Some problems that -Wextra will warn you about include:
- - Assert statements that always evaluate to true because of the datatype of the argument
- - Unused function parameters (only when used in conjunction with -Wall)
- - Empty if/else statements.

因此，最为强的验证是Werror选项，一般我们还是用-Wall多一些。

针对于本题的实现验证发现并没有出现warning错误，令人快慰。

之后，可以采用sanitizer来查看是否存在有内存方面的泄露。

- -fsanitize=address 

- - This flag enables the AddressSanitizer program, which is a memory error detector developed by Google. This can detect bugs such as out-of-bounds access to heap / stack, global variables, and dangling pointers (using a pointer after the object being pointed to is freed). In practice, this flag also adds another sanitizer, the LeakSanitizer, which detects memory leaks (also available via -fsanitize=leak).
- -fsanitize=undefined 

- - This flag enables the UndefinedBehaviorSanitizer program. It can detect and catch various kinds of undefined behavior during program execution, such as using null pointers, or signed integer overflow.
- -g flag 

- - This flag requests the compiler to generate and embed debugging information in the executable, especially the source code. This provides more specific debugging information when you’re running your executable with gdb or address sanitizers. You will see this flag being utilized in the next lab.

由于程序较为简短，并没有什么报错。

### 关于Makefile的使用
这里的教程写的很详细哦。

一些规则：

$@ represents the name of the current rule’s target.

$^ represents the names of all of the current rule’s dependencies, with spaces in between.

$< represents the name of the current rule’s first dependency.

注意这边提到的是current rule，在Makefile中要注意一些缩进上的艺术。比如要进行一次<tab>。