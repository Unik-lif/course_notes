## Intro:
> Why threads?

parallelism, more data sharing. (threads have their own stack, and the rest is the same)

> Disadvantages?

'atomicity' should be set to make the result deterministic. -> critical section and mutex. -> synchronization primitives.
## Thread API:
Always refer to the man page.

### For Threads: following steps.
To create a thread: pthread_create

To wait for a thread to complete: pthread_join
### For mutexs: following steps.
To create a mutex: pthread_mutex_t lock;

To initialize a mutex: 
1. pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER; -> static way.
2. if (pthread_mutex_init(&lock, NULL) == 0) -> dynamic way.

To lock a mutex: pthread_mutex_lock(&lock)

To unlock a mutex: pthread_mutex_unlock(&lock)

To destory a mutex: pthread_mutex_destroy()
### For Condition Variables.
condition variables are useful when some kind of signaling must take place between threads, If one thread is waiting for another to do something before it can continue.

two main functions:
1. int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
2. int pthread_cond_signal(pthread_cond_t *cond);

When compiling muti-thread programs, always use `-pthread`.

## Locks:
build locks yourself.


Basic categories:
- Test and Set
- Compare and Swap
- Fetch and Add

Lauer's Law: “If the same people had twice as much time, they could produce as good of a system in half the code.”

In the Above 3 methods, the spinning is inevitable. Maybe we should find a better way to avoid the cost. => Hardware support is not enough.

- yield the cpus
- queues: sleep instead of spinning
- - use park() to put a calling thread to sleep, and unpark(threadID) to wake a particular thread as designated by threadID

Look at the interesting code snippets here:
```C
typedef struct __lock_t {
    int flag;
    int guard;
    queue_t *q;
} lock_t;

void lock_init(lock_t *m) {
    m->flag = 0;
    m->guard = 0;
    queue_init(m->q);
}

void lock(lock_t *m) {
    // Spinning here to get into the doorway.
    while (TestAndSet(&m->guard, 1) == 1)
    ; //acquire guard lock by spinning
    if (m->flag == 0) {
        m->flag = 1; // lock is acquired
        m->guard = 0;
    } else {
        queue_add(m->q, gettid());
        m->guard = 0;
        park();
        // if we release guard lock after the park
        // the process will hold the lock while sleeping
        // other process won't proceed till untill the process wake up itself.

        // we just pass the lock directly from the thread releasing the lock to the next thread acquiring it
    }
}

void unlock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1)
    ; //acquire guard lock by spinning
    if (queue_empty(m->q))
        m->flag = 0; // let go of lock; no one wants it
    else
        unpark(queue_remove(m->q)); // hold lock
    // (for next thread!)
    m->guard = 0;
}
```
Finally, you might notice the perceived race condition in the solution,
just before the call to park(). With just the wrong timing, a thread will
be about to park, assuming that it should sleep until the lock is no longer
held. A switch at that time to another thread (say, a thread holding the
lock) could lead to trouble, for example, if that thread then released the
lock. The subsequent park by the first thread would then sleep forever
(potentially), a problem sometimes called the wakeup/waiting race.

Actually it is a problem, Solaris uses `setpark()` function to avoid it.

For Linux, use futex. Give each futex an address.
- futex_wait(address, expected)
- futex_wake(address)

## Lock DataStructures
Approximate counting:
- thread increments its local counter, periodically transferred the value to the global lock
- one way to do is set a threshold for the local counter

