## Hardware Description Languages and Verilog
Memory arrays can also perform boolean logic functions.

It is called Lookup Table(LUTs). LUTs are commonly used in FPGAS.

We can therefore easily transfer the bitstream into LUTs.

### verilog
more popular in US

### VHDL
more popular in Europe

### top-down design methodology
top-level->sub-module->leaf-cell

```verilog
module example(a, b, c, y);
    input a;
    input b;
    input c;
    output y;

//here comes the circuit description

endmodule

// or in this form

module test(input a, input b, output y);

endmodule
```
### Two Main styles of HDL implementation
#### Structural:
the module body contains gate-level description of the circuit.

##### just assume that you are building a circuit.
#### Behavioral:
the module body contains functional description of the circuit.

level of abstraction is higher than gate-level

basically we combine these two together.

##### use assign language to build circuits.
### Tri-state buffer
#### preface:
we can use Z to represent float.

X means I don't care what the value of this input is.

synthesize: draw a circuit graph

simulate: wave form diagram
#### more verilog examples:
at a low-level of abstraction: poor readability but more optimization

at a high-level of abstraction: great readability but less optimization

### Timing
only for simulation, cannot be synthesized.

used for modeling delays in a circuit.

timing is very important.
#### Good practice.
use a consistent naming style.

use MSB to LSB
```verilog
a[31:0]
```
define one module per file, easy to manage them together.

keep in mind that verilog describes hardware.
### For sequential Logic.
combinational + memory = sequential.

new constructs:
```verilog
always @ (sensitivity list)
    statement;
```
whenever the event in the sensitivity list happens, the statement is executed.
```verilog
always @ (posedge clk)
    q <= d;
```
### D flip-flop
```verilog
module flop(input clk, input [3:0] d, output reg [3:0] q);

always @ (posedge clk)
    q <= d;

endmodule
```
the name reg does not necessarily mean that the value is a register.(it could be, but it does not have to be).

### Asynchronous and Synchronous Reset
reset signals are used to initialize the hardware to a known state.

#### Asynchronous Reset
the reset signal is sampled independent of the clock.

#### Synchronous Reset
the reset signal is sampled  with respect to the clock.

