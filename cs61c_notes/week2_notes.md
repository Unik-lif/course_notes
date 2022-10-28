## Instructions:
- ### Goals of the computer designers:
    find a language that makes it easy to build the hardware and the compiler while maximizing performance and minimizing cost and energy.

- ### what we can learn from instructions:
    the secret of computing: the stored-program concept.

### Operands of the Computer Hardware
64 bits -> doubleword.

32 bits -> word.

- ### Alignment restriction: 

    words must start at addresses that are multiples of 4 and doublewords must start at addresses that are multiples of 8.

The process of putting less frequently used variables into memory is called spilling registers.

In RISC-V, the register x0 is hard-wired to be the value zero.

Java uses the first option to representing a string: the first position of the string is reserved to give the length of a string.

halfwords: 16-bit quantities.

### noted in Sept_11th:
A very annoying thing in IEEE754 standard: when representing denormal numbers, the exponent is implicitly -126, not -127.

That's standard with little reason but more consideration for use.

## Lab02:
1. ".c.o" is an old-fashioned suffix rule. better way to do this is: %.o : %.c. This means make any files stem.o from another file stem.c.
2. Shift unsigned for 32 times will cause great strange problem: as C programming language has puts, these things will be undefined. It won't behave well if you don't follow the standard, and the results will depend on the computer or architecture.
3. ~ will reverse all the bits. ! won't behave like this.
4. Why do both of these functions take in a node** and not a node*? Think about memory management; if the input was a node*, would it be possible to modify the pointer that was passed into the function from, say, the main() function? Remember that C is pass-by-value!
5. check test_main function.
### An interesting reverse.
```C
/* Reverse a linked list in place (in other words, without creating a new list).
   Assume that head_ptr is non-null. */
void reverse_list (node** head_ptr) {
	node* prev = NULL;
	node* curr = *head_ptr;
	node* next = NULL;
	while (curr != NULL) {
		/* INSERT CODE HERE */
		curr->next = next;
		curr->next = prev;
		prev = curr;
		curr = next;
	}
	/* Set the new head to be what originally was the last node in the list */
	*head_ptr = prev;/* INSERT CODE HERE */
}
```

### Draw picture to fix seemingly strange problems.
```
gdb-peda$ p curr
$2 = (node *) 0x7fffffffdde0
gdb-peda$ p curr->next
$3 = (struct node *) 0x555555555248 <main+117>
```

## Valgrind doc readings:
### usage for valgrind:
```shell
valgrind -leak-check=yes ./program [args] ;; enable detailed memory leak detector.
;; default function: memorycheck program.
```

some interesting facts:
1. --num-callers: if the stack trace is not big enough, use this option to make it bigger.
2. It's worth fixing errors in the order they are reported, as later errors can be caused by earlier errors. Failing to do this is a common cause of difficulty with Memcheck.
3. Memory leak messages look like these: "definitely lost" & "probably lost".
4. Memcheck also reprots unintialized values(Conditional jump or move depends on unintialized values), to get extra info using --track-origins=yes. ---> find where the unintialized values are coming from.

### Answers for lab01 -4
- Why didn’t the no_segfault_ex program segfault?

  cause it didn't overflow the range of the int array.
- Why does the no_segfault_ex produce inconsistent outputs?

  because it is stem from the wrong use of sizeof(a). This fuction will lead to result of 8, which is the same of the size of a pointer. Apparently, 8 exceeds 5, and therefore a[5] and other elements are undefined.

- Why is sizeof incorrect? How could you still use sizeof but make the code correct?

  we can simply use 5 instead.
### Valgrind for lab02
```
$ valgrind --tool=memcheck --leak-check=full --track-origins=yes [OS SPECIFIC ARGS] ./<executable>
```
using tool memcheck, enable leak-check, trace the origins to get extra info for unintialized values on some executable.

### RISCV-Details:
```
7.4 Which values need to saved by the caller, before 
jumping to a function using jal?

Registers a0 - a7, t0 - t6, and ra.

7.5 Which values need to be restored by the callee, 

before returning from a function?
Registers sp, gp (global pointer), tp (thread pointer), and s0 - s11. Note that we
don’t use gp and tp very often.

7.6 In a bug-free program, which registers are guaranteed to be the same after a function
call? Which registers aren’t guaranteed to be the same?

Registers a0 - a7, t0 - t6, and ra are not guaranteed to be the same after a function
call (which is why they must be saved by the caller). Registers sp, gp, tp, and s0 - s11 are guaranteed to be the same after a function call (which is why the callee
must restore them before returning).
```
An relative good summary for the registers in RISCV.
| Register(s) | Alt. | Description |
| ----------- | ---- | ----------- |
| x0 | zero | The zero register, always zero |
| x1 | ra | The return address register, stores where functions should return |
| x2 | sp | The stack pointer, where the stack ends |
| x5-x7, x28-x31 | t0-t6 | The temporary registers |
| x8-x9, x18-x27 | s0-s11 | The saved registers |
| x10-x17 | a0-a7 | The argument registers, a0-a1 are also return value |

## Lab03 notes:
It takes me a lot of time to read this .s code, but now I am able to analyse it. The comment is given here.
```S
.data # The data segement of this program. Under this scope we can define sth or some variables.
# Basically this segment will be stored in the .rodata part, that's the staic part of the program. Global variables are always stored here.
.word 2, 4, 6, 8 # an anoymous variable of 4 ints length is allocated here, so basically it will allocate some memory space.

n: .word 9 # we define a variable called n, it represents 9 as value. We can take n as the pointer, where it stores 9.

.text
main:   
    add t0, x0, x0
    addi t1, x0, 1 # t1 is 1.
    la t3, n # we load the address of n variable by simply calling n. This psedo-code can be seperated into two instructions.
    lw t3, 0(t3) # we load 9 into t3.
fib:    
    beq t3, x0, finish
    add t2, t1, t0 # t2 is 1. t2 is 2. t2 is 3.5.8.13.21.34.55
    mv t0, t1 # t0 is 1. t0 is 1. t0 is 2.3.5.8.13.21.34
    mv t1, t2 # t1 is 1. t1 is 2. t1 is 3.5.8.13.21.34.55
    addi t3, t3, -1  # t3 is 8. t3 is 7. t3 is 6.5.4.3.2.1.0
    j fib

finish: 
    addi a0, x0, 1
    addi a1, t0, 0
    ecall # print integer ecall
    addi a0, x0, 10
    ecall # terminate ecalls
```
Question parts:
1. What do the .data, .word, .text directives mean (i.e. what do you use them for)? Hint: think about the 4 sections of memory.
```
.data: this is the data segment, it starts series of variable declarations.
.word: directive allocates memory.
.text: actual code will be put into this part. When assembler see the .text, it will switch to the text segment, and it is where the code goes.
```
2. Run the program to completion. What number did the program output? What does this number represent?
```
the result is 34. It represents the 9th fibonacci number.
```
3. At what address is n stored in memory? Hint: Look at the contents of the registers.
```
n is stored at 0x100000000 + 8. 

extra task when doing the HW: should be done by the end of this week.

configure a toolchain for riscv to working in linux environment, therefore we can get the elf and check the elf to check the understanding of below message:

**gp is always .data + 0x800 in RISC-V 32 architecture. And gp will make sure that all of the gloabl variables is in the scope +/-2KiB range.**
```
4. Without actually editing the code (i.e. without going into the “Editor” tab), have the program calculate the 13th fib number (0-indexed) by manually modifying the value of a register. You may find it helpful to first step through the code. If you prefer to look at decimal values, change the “Display Settings” option at the bottom.

```
simply changing the value of the t0 and t1 would suffice our demands. the result is 0x90, which is 144 in decimal.
```

### lab3ex2.
1. The register representing the variable k.

   This register is t0.
2. The register representing the variable sum.
   
   s0.
3. The registers acting as pointers to the source and dest arrays.

   source: s1.
  
   dest: s2.
4. The assembly code for the loop found in the C code.
    
    trivial.
5. How the pointers are manipulated in the assembly code.

   get the true address -> using lw/sw instructions.
   
A recursive function in assembly be like:
```assembly
function:
  addi sp, sp, (some immediate number)
  sw....
  sw....
  #(trivial case)
  beq...go straight to 'done'.
  #(change argument registers)
  jal ra, function
done:
  lw...
  lw...
```

### Some conventions:
1. 'ret' is equal to 'jalr x0, x1, 0', as x1 is equal to register 'ra', this simply jump back to the caller.
2. 'beq', branch instruction always want to find the address label in the file itself, while 'jar' can more easily jump out of the file.
3. 'la' is a psedo code that used for some immediate address.
4. Mind 'ra' when calling callee functions, better to store it. In convention, we always get the ra on the stack, only by optimization will we sometimes simply never touch this register. Keep in mind to have a closer look if we have used jump or other instructions related to 'ra' register.
5. Using temporary register might not always be a good idea, sometimes it will change the temporary register value in the caller function. Therefore, leading to nasty change of values.
6. local values should always be put into the stack.caller里面用save callee里面用temporary。（妈的这下真就是血淋淋的教训了）