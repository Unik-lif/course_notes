## lec1: intro
Therac-25: a radiation therapy machine, injured patients died from lethal dosage of radiation caused by a software bug.

### system & complexity
system: interacting set of components with a specified behavior at the interface with its environment

a systematical way to understand a complex system is needed.

we get 14 properites of computer systems:
1. correctness
- every third bug is not a bug, it's not a bug, it's a feature.
- how to describe the functionalities of your code?
2. latency
3. throughput
4. scalability
5. utilization
6. performance isolation
- see 'Perfiso: performance isolation for commercial latency-sensitive services'.
7. energy efficientcy
8. consistency
9. fault tolerance
- cosmic radiation bug.
- rowhammer attack.
10. security
- cold-boot attack
11. privacy
- privacy mode is not so powerful as described.
12. trust
13. compatibility
- itanium, not compatible to x86, died.
14. usability
- iphone

These properties have some confflicts. => all these lead to more complexity.

Problems:
1. Emergent properties => properties are not considered at design time.
2. Propagation of effects => butterfly effort.
3. Incommensurate scaling => design for small model may not scale.
4. Trade-offs => can't sell the cow and drink the milk.