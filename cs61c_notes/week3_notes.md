## lab4
### some problems:
- Why do we need to save stuff on the stack before we call jal?

because we need ret to jump back to the address stored in ra.

- What’s the difference between add t0, s0, x0 and lw t0, 0(s0)?

the former one simply add the number, but lw will fetch the true thing that s0 will point to.

- Pay attention to the types of attributes in a struct node.

the type should be taken care of with the length of bytes.

- Note: you need only focus on map, mapLoop, and done functions but it’s worth understanding the full program.
- Note: you may not use any s registers outside of s0 and s1.

lab4 is easy, much easier than proj2.
