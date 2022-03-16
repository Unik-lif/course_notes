## Circuit Timing
timing:
1. how fast is a circuit?
2. how can we make a circuit faster?
3. what happens if we run a circuit too fast?
### Part1: combinational logic
transistors take a finite amount of time to switch

caused by capacitance and resistance in a circuit.

this delay can be seperated into two parts:
1. contamination delay: $t_{cd}$, delay until Y starts changing: shortest delay
2. Propagation delay: $t_{pd}$, delay until Y finishes changing: longest delay

we care both the longest and shortest delay paths in a circuit.

the delay of these circuits highly depends on the tempreture and the voltage.

#### Part1.5: Output Glitches:
one input transitions causes multiple output transitions.

##### avoid it!
1. use K-Maps to build better logic circuit.
2. however, if we only care about the long-term steady state output, we can safely ignore glitches.

### Part2: D Flip-Flop input timing constraints
D must be stable when sampled.

1. setup time: time before the clock edge that data must be stable
2. hold time: time after the clock edge that data must be stable
3. aperture time: time around clock edge that data must be stable.是上述两个时间之和。

due to the internal sampling mechanism, the input should be stable.

if D is changing when sampled, metastability can occur.

flip-flop output is stuck somewhere between '1' and '0'.

the output  timing model is shown below.

if we want to design a sequential system.

this means there is both a minimum and maximum delay between two filp-flops.

