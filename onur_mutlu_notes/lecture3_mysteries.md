## DRAM

dram cell consists of a capacitor and an access transistor.

capacitor. It stores data in terms of charge status of the capacitor.

So apparently with time comes the state change.

DRAM capacitor charge leaks over time.

so basically the dram should refresh.

typical N = 64ms, when temperature goes high, the frequency should also be higher.

More energy should be used to fresh the dram.

### question.
observation: all dram rows are refreshed every 64ms

critical thinking: do we need to refresh all rows every 64ms?

a recent paper discover that, only a very slim part of dram should refresh per 64ms.

![photo1]()

Not all cells are exactly the same, and some are leakier than others.

This is called manufacturing process variation.

### RAIDR:
1. identify the retention time of all dram rows.
2. store rows into bins by retention time.

## Memory performance:
multi-core systems.

simpler and lower power than a single large core.

we predicted to have linear scalable rise, but in reality we failed.

Unexpected slowdowns in multi-core.

problem:
1. why is there any slowdown?
2. why is there a disparity in slowdowns?
3. how can we solve the problem if we do not want that disparity?

consolidate and mix.

### memory performance attack.
unfairness.

同时使用stream和random两种内存阅读方式，

