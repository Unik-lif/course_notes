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
### CASES:
#### interrupts:
- pros: easy
- negs: 1. no need for privilege; trust this facility is not abused. 2. multiprocessor faults: can't fit in this case. 3. lost of interrupts.