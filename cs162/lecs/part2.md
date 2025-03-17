## Concurrency
## Lec 6
Client is somtimes on, while the server is always on.

A connection between two endpoints A and B consists of:
- A queue for data sent from A to B
- A queue for data sent from B to A

Key idea: using Socket abstraction as the endpoint for communication, communication across the wolrd looks like File I/O

socket is a bi-directional communication method.

> However, the processes are seperate, how do they know they want to interact?

using namespaces for communication over IP

### Connection setup over TCP/IP
Server side
- create server socket
- bind it to an address (host:port)
- listen for connection
- accept syscall
- create another socket to the client
- read request
...

Client side
- create client socket
- connect it to server (host:port)
- write request 
...
### concurrency
that is about scheduling => about queues

Multiple Queue, Multiple policy
## Lec 7
threads switch will be easier.

Linux number:
- frequency of context switch: 10-100ms
- switching between processes: 3-4 usec
- switching between threads: 100 ns

### user level and kernel level thread
user level switch => only a function call to the library.

### Simultaneous MultiThreading/Hyperthreading
superscalar processors can execute multiple instructions that are independent

hyper-threading duplicates register state to make a second 'thread'

The key idea: a stream doesn't use all of the components at all times, we can then support another thread using the remaining components.

This gives one core the illusion of two core, which 'looks like' multiprocessor. 

However, due to its method, some conflicts might need time to be handled correctly, so we will have non-linear acceleration.
### Interrupt and threads
save states, disable and reenable interupt, restore states and go back.

### Thread usage
Each work can be assigned to one thread.

But we should have atomic operation to make it indivisible, or the number and behaviour will go wild.

### Synchronization
#### Lock
less interesting

Maybe I need refer to the book now!! I will open another directory here record what I've read.-