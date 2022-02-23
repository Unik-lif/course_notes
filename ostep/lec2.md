## Learn about CPU virtualization
two piece:
1. mechanisms
2. policies (scheduler)

Background:
```
 _ _ _ _        _ _ _ _ _ _
|       |      |           |
|  CPU  |      |   Memory  |
 - - - -        - - - - - -
    |_ _ _ _ _ _ _ _ |

while(1) {
    fetch(pc)
    decode: Figure out which instruction is
    (inc. PC)
  - - - -[execute: (could change PC)]    (process Interrupts)->handle pending interrupts => OS gets involved.
|}                                     
|_ _ _ _ before execute, check permission. 
          modes: user & kernel) is it OK to execute this instruction? 
          (if not OK, raise exception(OS involved))
```

CPU virtual mechanisms:

OS should be a limited Direct Execution.

procedure:
1. Boot time: OS runs first (privileged "kernel" mode)
2. install handlers (code)
3. tell hardware what code to run on exception / interrupt / traps.(done by privileged instruction)
4. init timer interrupt.
5. Ready to run user programs.

if a user program A wants OS service (system call)

issue special instructions:

    trap instruction (x86 called "int")
    transition from user mode to kernel mode.
    jump into OS: 
        target: trap handlers
    save register state (so as to enable resume execution later).

Os sys call handler: runs
    => ret-from-trap (opposite of about)