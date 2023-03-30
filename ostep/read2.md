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
## Scheduling: The Multi-Level FeedBack Queue
MLFQ: Multi-Level Feedback Queue.

Fundamental Problems:
1. OS doesn't generally know how long a job will run for.
2. minimize the response time.

Basic Rules:
1. If Priority(A) > Priority(B), A runs.
2. If Priority(A) = Priority(B), A & B run in RR.

### Attempt #1: how to change priority
3. When a job enters the system, it is placed at the highest priority.
4. If a job uses up an entire time slice while running, its priority is reduced.
5. If a job gives up the CPU before the time slice is up, it stays at the same priority level.

### Problems:  
1. Starvation: if there are "too many" interactive jobs in the system, long-running jobs will never receive any CPU time.
2. Game the scheduler: near the end of the time slice, trigger an I/O operation.
3. a program may change its behavior over time.

Note that scheduling policy forms an important part of the security of a system, and should be carefully constructed.

### Attempt #2: Priority Boost
6. After some time Period S, move all the jobs in the system to the topmost queue.

the Value of S is kinda voo-doo constants. Like Black Magic. (Ousterhout's Law)

### Attempt #3: Better Accounting
Not simply detect whether the task has used all the time slice or not, but accounting the total time. Let the Scehduler keep track.

Revised Version of Rule4 & Rule5:
#### Rule 4: Once a job uses up its time allotment at a given level, its priority is reduced.

### Other Issues:
1. How to Parameterize the scheduler?

varying time-slice length across different queues, high-priority queues -> short time slices, low-priority queues -> long time.

In a system, there is a default parameter sets table.

Other Ideas: **decay-usage algorithms**

### HW:
1. Run a few randomly-generated problems with just two jobs and two queues; compute the MLFQ execution trace for each. Make your life easier by limiting the length of each job and turning off I/Os.
```
link@ubuntu:~/Desktop/ostep-homework/cpu-sched-mlfq$ python3 mlfq.py -s 40 -n 2 -q 1 -j 2 -m 5 -M 0 -c
Here is the list of inputs:
OPTIONS jobs 2
OPTIONS queues 2
OPTIONS allotments for queue  1 is   1
OPTIONS quantum length for queue  1 is   1
OPTIONS allotments for queue  0 is   1
OPTIONS quantum length for queue  0 is   1
OPTIONS boost 0
OPTIONS ioTime 5
OPTIONS stayAfterIO False
OPTIONS iobump False


For each job, three defining characteristics are given:
  startTime : at what time does the job enter the system
  runTime   : the total CPU time needed by the job to finish
  ioFreq    : every ioFreq time units, the job issues an I/O
              (the I/O takes ioTime units to complete)

Job List:
  Job  0: startTime   0 - runTime   2 - ioFreq   0
  Job  1: startTime   0 - runTime   1 - ioFreq   0


Execution Trace:

[ time 0 ] JOB BEGINS by JOB 0
[ time 0 ] JOB BEGINS by JOB 1
[ time 0 ] Run JOB 0 at PRIORITY 1 [ TICKS 0 ALLOT 1 TIME 1 (of 2) ]
[ time 1 ] Run JOB 1 at PRIORITY 1 [ TICKS 0 ALLOT 1 TIME 0 (of 1) ]
[ time 2 ] FINISHED JOB 1
[ time 2 ] Run JOB 0 at PRIORITY 0 [ TICKS 0 ALLOT 1 TIME 0 (of 2) ]
[ time 3 ] FINISHED JOB 0
```
2. How would you run the scheduler to reproduce each of the examples in the chapter?
```
skip
```
3. How would you configure the scheduler parameters to behave justlike a round-robin scheduler?
```
simply set -n=1
```

## Scheduling: Proportional Share
Instead of optimizing for turnaround or response time, a scheduler might instead try to guarantee that each job obtain percentage of CPU time.

### Lottery scheduling:
Basically it acts like a random algorithm. For user A and user B, they have 25 and 75 tickets, which is a certificate for their chances of gaining resources next time.

#### Ticket Mechanism:
1. ticket currency: just look at the account.
2. ticket transfer: transfer my tickets to the user I want to act some operations at once.
3. ticket inflation: change my own ticket number when facing different tasks. (for boosting or other purpose)
#### Implementation
```C
int counter = 0;
int winner = getrandom(0, totaltickets);
node_t *current = head;
while (current) {
  counter = counter + current->tickets;
  if (counter > winner) 
    break;
  current = current->next;
}
```
Lottery goals: form a perfectly fair scheduler, which would achieve F = 1.

F is simply the time the first job completes divided by the time that the second job completes.

From the figure 9.2, we can see that when job length is very long, the fairness will be very close to 1.
### Stride Scheduling:
As we saw above, while randomness gets us a simple scheduler, it occasionally will not deliver the exact right proportions, especially over short time scales. That's the reason why we invented stride scheduling.

Stride scheduling:
1. compute each job's stride. (1 / tickets)
2. find the job that has the smallest progress so far.
3. so on.

```c
curr = remove_min(queue); // pick client with min pass
schedule(curr); // run for quantum
curr->pass += curr->stride; // update pass using stride
insert(queue, curr); // return curr to queue
```
Stride has shown very good fairness, so why we still use Lottery scheduling sometimes?

reasons are easy. When a new job enters, it will monopolize the CPU, which will do lots of harm.
### Linux Completely Fair Scheduler
CFS spends little time making scheduling decisions.
#### Basic Operation
Using **virtual runtime(vruntime)**. For each process, accumulates `vruntime`. The `vruntime` increases at the same rate, in proportion with physical time. CFS will always pick the process with the lowest vruntime to run next.

**Tension:** when switches happen quite often, the fairness will be increased. Otherwise, the preformance is increased when switches happens less often.

`sched_latency` is a parameter to determine how long one process should run before considering a switch.

$Timeslice = schedlatency / n, n = processesnumber$

`min_granularity` is usually set to a value like 6 ms. CFS will never set the time slice of a process to less than this value, ensuring that not too much time is spent in scheduling overhead.(to avoid too many processes dividing sched_latency into too small pieces)

#### Weighting(Niceness)
The priority: -20 ~ +19. With a default of 0.

negative values imply higher priority. If you are too nice, you just don't get as much scheduling attention.
```C
static const int prio_to_weight[40] = {
  /*-20*/ 88761, 71755, 56483, 46273, 36291,
  /*-15*/ 29154, 23254, 18705, 14949, 11916,
  /*-10*/  9548,  7620,  6100,  4904,  3906,
  /*-5*/  3121,  2501,  1991,  1586,  1277,
  /*0*/  1024,   820,   655,   526,   423,
  /*5*/   335,   272,   215,   172,   137,
  /*10*/   110,    87,    70,    56,    45,
  /*15*/    36,    29,    23,    18,    15,
};
// roughly 3 times multi for weights that have difference equals 5. 
```
The weights allow us to compute the effective time slice of each process. Assume here we have n processes.
$$time\_slice_k = \frac{weight_k}{\sum^{n-1}_{i = 0} weight_i} * sched\_latency$$

#### Using Red-Black Trees
Having 1000 processes running at the same time is a very common situation. So apparently using list to manage is a nightmare.

CFS addresses this by keeping processes in a red-black tree. Only running processes are kept therein. If a process goes to sleep, it is removed from the tree and kept track of elsewhere.

O(log N) hahaha. for searching/removing/inserting etc.
#### Dealing with I/O and sleeping Processes
For jobs that have slept for a long time, to avoid monoplizing the CPU, CFS will set the vruntime of that job to the minimum value found in the Red-Black tree.

However, that means the job slept for a while won't be treated fairly. That's the cost.

#### HW:
fairly easy. We skip it.