刚刚从北京回家，很累，今天开摆一天，简单看一看交大的`PPT`学习一下。
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
- speed up of Ethernet, some changes to detect collision.
2. Propagation of effects => butterfly effort.
3. Incommensurate scaling => design for small model may not scale.
- different parts of the system exhibit different orders of growth.
4. Trade-offs => can't sell the cow and drink the milk.
### Handle Complexity:
cases studies.

systems are indefinitely: Limits the depth of digging.

Some methods:
1. Modularity: split up system
2. Abstraction: Interface/Hiding
3. Layering: gradually build up capabilities
4. Hierarchy: Reduce connections, divide & conquer

## lec2: Scalability in Practice
In modern Life, each click on the website needs thousands of servers to cooperate.

Build a website on one machine:
```
https://www.taobao.com => Internet ====> Application ==> Database: user, price
                                              |
                                              | =======> File: Image
```
Limitations:
1. The disk & memory of one server => can't store massive amount of data.
2. Applications uses the CPU for processing, while a single CPU can hardly scale due to the end of **Moore's Law & Dennard Scaling**.
- Dennard scaling: as transistors get smaller, their power density stays constant.

To scale:
1. disaggregating application & data.
- more CPUs to process application.
- more disk & cache to maintain database.
- more disks to store large bulk of data for the File system
2. Caching: avoid slow data accesses.
- distributed caching server.
- [consistent hasing to find the server to cache data.](https://www.bilibili.com/video/BV1q4411w7Dw?p=2&vd_source=49f5b184846e52adee8a1a5165c5f962)
3. more servers: for stateless application servers.
- only executes the logic relies on input data.
- stateless servers can have better fault-tolerance and better elasticity.
4. scaling database
- separate the database for read/write
- separate a table on multiple databases
5. using distributed file system
6. using CDN.
- content delivery network caches the content at the network providers, which is closer to users.
7. separate different applications
- dedicated servers for different applications.