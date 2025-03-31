# Project 0: Getting Real

## Preliminaries

>Fill in your name and email address.

Linke Song <songlinke@iie.ac.cn>

>If you have any preliminary comments on your submission, notes for the TAs, please give them here.



>Please cite any offline or online sources you consulted while preparing your submission, other than the Pintos documentation, course text, lecture notes, and course staff.



## Booting Pintos

>A1: Put the screenshot of Pintos running example here.

```
root@83a3033c59ed:~/pintos/src/threads# pintos --                 
qemu-system-i386 -device isa-debug-exit -drive format=raw,media=disk,index=0,file=/tmp/7C0hMe0Rsw.dsk -m 4 -net none -nographic -monitor null
Pintos hda1
Loading...........
Kernel command line:
Pintos booting with 3,968 kB RAM...
367 pages available in kernel pool.
367 pages available in user pool.
Calibrating timer...  157,081,600 loops/s.
Boot complete.
```

```
root@83a3033c59ed:~/pintos/src/threads# pintos --bochs --
squish-pty bochs -q
========================================================================
                       Bochs x86 Emulator 2.6.2
                Built from SVN snapshot on May 26, 2013
                  Compiled on Mar  1 2022 at 16:09:16
========================================================================
00000000000i[     ] reading configuration from bochsrc.txt
00000000000e[     ] bochsrc.txt:8: 'user_shortcut' will be replaced by new 'keyboard' option.
00000000000i[     ] installing nogui module as the Bochs GUI
00000000000i[     ] using log file bochsout.txt
Pintos hda1
Loading...........
Kernel command line:
Pintos booting with 4,096 kB RAM...
383 pages available in kernel pool.
383 pages available in user pool.
Calibrating timer...  102,400 loops/s.
Boot complete.
```
## Debugging

#### QUESTIONS: BIOS 

>B1: What is the first instruction that gets executed?



>B2: At which physical address is this instruction located?




#### QUESTIONS: BOOTLOADER

>B3: How does the bootloader read disk sectors? In particular, what BIOS interrupt is used?



>B4: How does the bootloader decides whether it successfully finds the Pintos kernel?



>B5: What happens when the bootloader could not find the Pintos kernel?



>B6: At what point and how exactly does the bootloader transfer control to the Pintos kernel?



#### QUESTIONS: KERNEL

>B7: At the entry of pintos_init(), what is the value of expression `init_page_dir[pd_no(ptov(0))]` in hexadecimal format?



>B8: When `palloc_get_page()` is called for the first time,

>> B8.1 what does the call stack look like?
>>
>> 

>> B8.2 what is the return value in hexadecimal format?
>>
>> 

>> B8.3 what is the value of expression `init_page_dir[pd_no(ptov(0))]` in hexadecimal format?
>>
>> 



>B9: When palloc_get_page() is called for the third time,

>> B9.1 what does the call stack look like?
>>
>> 

>> B9.2 what is the return value in hexadecimal format?
>>
>> 

>> B9.3 what is the value of expression `init_page_dir[pd_no(ptov(0))]` in hexadecimal format?
>>
>> 



## Kernel Monitor

>C1: Put the screenshot of your kernel monitor running example here. (It should show how your kernel shell respond to `whoami`, `exit`, and `other input`.)

#### 

>C2: Explain how you read and write to the console for the kernel monitor.