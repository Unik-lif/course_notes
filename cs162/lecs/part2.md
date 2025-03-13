## Concurrency
## Lec 6
Client is somtimes on, while the server is always on.

A connection between two endpoints A and B consists of:
- A queue for data sent from A to B
- A queue for data sent from B to A

Key idea: using Socket abstraction as the endpoint for communication, communication across the wolrd looks like File I/O

socket is a bi-directional communication method.

> However, the processes are seperate, how do they know they want to interact?

30:51