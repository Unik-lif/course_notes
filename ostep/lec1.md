# CS537 operating system 
## Lec1 Part1.
### Why study OS?
stud: security, apps/portability, reliability, file/program.

tea: learn how computers work? and interesting, but there's so much more, we had to take it.
### background:
how computer works?

#### basic idea.
```
cpu <-> Mem

fetch decode execute
* get instructions from mem
* figure them out
* execute them

low-level -> C program level.

should know:
(code heap stack)
```
os: what is it?

os illusion for memory:

set boundary automatically for different programs, even if they look like that they use the same address of memory. So you don't need to worry about whether another program occupy the same address as your programs do.

## Lec1 Part2
course info
## Lec1 Part3
try to understand details.
### Virtualization
taking one physical thing into many virtual ones.

it is really an illusion. Each running program thinks that it has its own cpu and own private memory.

#### key aspects:
1. efficient.
2. secure(restricted user's privileged range).

CPU virtualization:
=>time-sharing.

basic idea about time-sharing: A | B | C | A | B | C

Abstraction: process. ~ running program.

(components of a process)
```
process---> registers.(pc etc).
       |_ _ (address space) code heap stack.
       |_ _ I/O states.
 _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
|                              |
|  process {memory(private)}   |
|                              |
 - - - - - - - - - - - - - - --
```
#### Mechanisms:
How can we do like things mentioned above?

Based on Policy.

#### the core mechansisms:

#### limited direct Execution
1. limited: security(protection).
2. direct Execution: efficiency, as fast as possible.

programs that do as Direct Execution:

OS: the first program to run in the machine.

want: run program 'A' as Direct Execution.

steps:
1. load 'A' from disk into memory.
2. CPU: set up 'sp' to stack, 'pc' to code.

procedure:
```
OS --->
       \
        \--> A ---> run
Problems:
1. what if 'A' process(want) to do something restricted ?
2. what if os wants to stop 'A' and run 'B' ?
3. what if 'A' does something that is slow ?
```

#### mode: 

=> OS: kernel mode. can do anything.

=> user program: user mode. can only do limited of things.

Question:

how to get into these modes?

how to transition?
```
@boot time:
1. boot in kernel mode.

2. want to run user program.
    1. transition into user mode.
    2. jumps to some location in user program.
3. wants to do something restricted.
    1. change to kernel mode.
    2. jumps into kernel but restricted jump(can't allow jump to anywhere). 
```
so how can a user mode changed to kernel mode, execute priviledged procedure, and return?
1. jump into kernel
2. elevate privilege
3. save enough register state so that we can return properly
```
A -->          ---> 
     \        /
  trap\      /ret_from_trap
       OS -->
    (system calls)

@boot time: OS
    => kernel mode
    => set up trap handlers (by issuing special instruction: tell hardware where trap handlers are in OS memory)

```
save / restore process:

use trap handler to manage trap.

