## Scheduling: Introduction
low-level mechanisms -> high-level policies
### metrics for scheduling:
$T_{turnaround} = T_{completion} - T_{arrival}$ -> Performance.

Another metric is fairness.
### Strategies
#### FIFO: First In, First Out. 
Or, First Come, First Serve. (FCFS)

**Bad case:** This scheduling scenario mightremind you of a single line at a grocery store and what you feel like whenyou see the person in front of you with three carts full of provisions anda checkbook out.
#### SJF: Shortest Job First
**Bad cases:**: Jobs not come at once, which may be harmful if you use non-preemptive strategy.
#### STCF: Shortest Time-to-Completion First
Or, PSFJ(Preemptive Shortest Job First).

This is a preemptive startegy.
### New Metirc: Response Time
**Realistic Demands**: Now users would sit at a terminal and demand interactive performance from the system as well.

**Demands for less starvation.** 

-> **response time**:

$T_{response} = T_{firstrun} - T_{arrival}$

-> Round-Robin

#### Amortization: let the cost be more acceptable when there is a fixed cost to some operations.
For example, if the time slice is set to 10 ms, and thecontext-switch cost is 1 ms, roughly 10% of time is spent context switch-ing and is thus wasted. If we want toamortizethis cost, we can increasethe time slice, e.g., to 100 ms. In this case, less than 1% of time is spentcontext switching, and thus the cost of time-slicing has been amortized.

Therefore, the time-slice of RR should be carefully chosen.

#### Disadvantage of RR.
When tasks have close time to finish, RR will be very bad for turnaround time.
### I/O corporation
Better Ovlapping: with the CPU being used by one process while waiting for the I/O of another process to complete, therefore the system is thus better utilized.

### Remain tasks:
In this chapter, we have the priori knowledge for how long the task is, this is unrealistic in reality.

### HW:
1. Compute the response time and turnaround time when runningthree jobs of length 200 with the SJF and FIFO schedulers.
```
Wait Time: any time spent ready but not running
For FIFO:
./scheduler.py -p FIFO -j 3 -l 200,200,200 -c

Turnaround  Response    Wait
200         0           0
400         200         200
600         400         400
--------------------------------------------------
For SJF:
The same.
```
2. Now do the same but with jobs of different lengths: 100, 200, and 300.
```
For SJF:
./scheduler.py -p SJF -j 3 -l 100,200,300

Turnaround  Response    Wait
100         0           0
300         100         100
600         300         300
--------------------------------------------------
For FIFO:
It depends on the `-l` order.
```
3. Now do the same, but also with the RR scheduler and a time-slice of 1.
```
Skip. -> just know the turnaround time is truly so long, and the response time is truly so short.
```
4. For what types of workloads does SJF deliver the same turn around times as FIFO?
```
If the Shortest Job comes first (from shortest to longest, in order), then these two will be the same.
```
5. For what types of workloads and quantum lengths does SJF deliver the same response times as RR?
```
Jobs are of the same length, and this length is also equal to quantum length.
```
6. What happens to response time with SJF as job lengths increase? Can you use the simulator to demonstrate the trend?
```
The time will be longer.
Skip.
```
7. What happens to response time with RR as quantum lengths in-crease? Can you write an equation that gives the worst-case response time, given N jobs?
```
The time will be longer.
T_{worst_response} = (N - 1) * T_{quantum}
```