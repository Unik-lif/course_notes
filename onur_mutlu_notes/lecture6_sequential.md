## Main Theme: Elements that can store information
key Idea:

We want circuits that produce output depending on current and past input values - circuits with memory.

### Capturing Data
#### cross-coupled inverters

### Memory
![photo1](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-26%20213801.png)
### Sequential Logic Circuits
combinational: only depends on current inputs.

sequential: opens depending on past inputs.

use state machine, under this state machine transition graph conditions can we get the state we needed.

#### clock cycle: depending on your machine ability.
### Finite State Machine
a discrete-time model of a stateful system.

consist of five elements:
1. finite number of states
2. finite number of inputs
3. finite number of outputs
4. explicit specification of all state transitions
5. explicit specification of what determines each external output value

Each FSM consists of three separate parts:
1. next state logic
2. state register
3. output logic

can be demostrated like below：
![photo2](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-26%20223402.png)

### Mind basic models:
R-S latch & D Flip-Flop

R-S latch is the basic model that can store value, but it always change when CLK is 1, therefore an anoiying thing happen like below:
![photo3](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-26%20225106.png)
Red circle is what we don't want.

there comes D Flip-Flop
1. state change on clock edge.
2. data available for full cycle.
![photo4](https://gitee.com/song-linke/photos/raw/master/Screenshot%202022-01-26%20225730.png)

主要想法：
1. 在CLK为0时Q保持不变。
2. 在CLK为1时Q从D latch(master)处获得输入，该输入被D锁存器锁存着，仅存储了CLK为0时末端时刻的状态，在上升沿真正来临时，Master锁存器被锁住，这个值不能受输入D影响。
#### 从而实现了一定程度上的滤波效益。

### FSM state encoding:
#### binary encoding:
States:00,01,10,11 etc.
#### One-Hot encoding:
each bit encodes a different state
#### output encoding:
001,010,100,110: 110 means yellow plus red

### Moore vs Mealy Machines:
Moore Fsm: outputs depend only on the current state

Mealy Fsm: outputs depend on the current state and the inputs.

