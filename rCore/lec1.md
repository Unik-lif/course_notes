## Lec1:
### Importance of RTFM.
#### main focus:
PPT & Volume I & Volume II.
### PPT infos:
#### RISC-V arch:
RISC-V -> many architectures with one ISA set.
#### Modes:
M: Machine mode -> for every risc-v chip, M-mode is necessary.

S: System mode -> OS.

U: User mode -> Apps.

SBI: a convention in RISC-V used in M-mode to let the OS to call some instructions.
#### Mode Specifics CSRs.
CSRs have their own address space. -> Not similar to regular registers.

Each hart(CPU core) has its own set of 4K CSRs. memory address (1K/mode, we have 4 modes).
### Interrupts vs Exceptions.
xTVEC: CSR that holds handler address.
- two strategys for jumping: 
1. jump to a fixed function handler. 
2. jump according to a IDT.

xEDELEG: CSR select mode to trap into (which mode).

xTVAL: CSR saves additional information about cause.

xEPC: CSR saves return Program Counter.

xSTATUS: CSR saves current Mode.

#### Distinguish:
Whether MSB is equal to 1 or not.

### Chinese manuals:
Refer to P102 in the chapter 10. Great importance.

mscratch -> great importance.