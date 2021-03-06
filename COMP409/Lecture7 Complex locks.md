# Lecture 7. 01.31 Complex locks
## Java's Locks (synchronous)
* 如果我们把monitor_enter/monitor_exit看成是Fat Lock方式，则可以把Thin Lock看成是一种基于CAS（Compare and Swap）的简易实现。
* 而基于CAS方式的实现，线程进入竞争状态的，获得锁的线程，会让其他线程处于自旋状态（也称之为Spin Mode，即自旋），这是一种while(Lock_release) doStuff()的Busy-Wait方式，是一种耗CPU的方式；而Fat Lock方式下，一个线程获得锁的时候，其他线程可以先sleep，等锁释放后，再唤醒（Notify）。
* CAS的优点是快，如果没有线程竞争的情况下，因为CAS只需要一个指令便获得锁，所以称之为Thin Lock，缺点也是很明显的，即如果频繁发生线程竞争，CAS是低效，主要表现为，排斥在锁之外的线程是Busy Wait状态；而monitor_enter/monitor_exit/monitor_notify方式，则是重量级的，在线程产生竞争的时候，Fat Lock在OS mutex方式下，可以实现no busy-wait。
* "Thin" Lock (Bacon lock)
    * Optimized lock, contention is rare
    * Each object has lock word
        * Divided into 8, 8, 15, 1 bits
        * First 8 bits is reserved （没有用到）
        * Second 8 bits - recursive lock count -1 （第一次不需要显示1）
        * 15 bits represent thread id of owner
            * 0 represents no thread, and consequently no lock
        * Last 1 bit = lock shape - 0 = cheap (thin); 1 = expensive (fat) lock
    * Lock done using `CAS(lockword, 0, id << IDOFFSET)`
        * true &rarr; we own it &rarr; locked
        * false &rarr; check the shape bit, verify it is a 0(i.e. thin lock)
            * false &rarr; check ownership
                * ownership: true &rarr; locking again, add 1 to lock count. Make sure there is no overflow for the count
        * If fail occurs anywhere, transition into "fat lock"（变成flat school）
            * Transition to fat lock is done by user.
            * If we don't own it, we spin until we do, or until it becomes a fat lock (known from shape bit)

* "Fat" Lock
    * Need mutex
    * Lock divided into 8, 23, 1
    * 23 bit - points to the pointer to mutex 
    * 1 bit - shape bit (1)
* 如果T1, T2，T3，T4...产生线程竞争，则T1通过CAS获得锁(此时是Thin Lock方式)，如果T1在CAS期间获得锁，则T2，T3进入SPIN状态直到T1释放锁；而第二个获得锁的线程，比如T2，会将锁升级（Inflation）为Fat Lock，于是，以后尝试获得锁的线程都使用Mutex方式获得锁。
* Tasuki锁为这种方式做了2个优化：
    * avoid spinning as much as possible
    * allow fat lock "deflation" (退化) back to thin lock
    * uses a little more space

---
## Blocking 
* So far &rarr; mostly spin-locks
* Drawbacks
    * spinning uses CPU
    * for longer spinning situation, it better if we don't need schedule at all.
    * Solution:
        * Instead of spinning, we go to sleep();
        * Intention &rarr; whoever leaves the lock wakes us up
            * T0: 
            if (locked) {
                // GAP!!!
                sleep();
            }
            * T1:
            unlock();
            wakeup();
            * Drawback: easy to lose a wake up ("lost wakeup" problem)
---            
## Semaphore & Mutexer

Semaphore
* Blocking synchronize &rarr; sleep & wake
* Abstraction
* Semaphone value: Integer (>=0)
* P(S) "down" (all atomic)
    ```java
    // atomic
    while (s == 0) {
        sleep();
    }
    s--;
    ```
* V(s) "up" (all atomic)
    ```java
    // atomic
    s++;
    wakeup();    // call some wakeup
    ```
Q: Who gets woken up?
A: Arbitrary

Binary Semaphore
* Always 0 or 1
* Starts at 1
    * lock = P()
    * unlock = V()
Q:What if our binary semaphore start at 0 ?
A:一个线程运行先被阻断，另一个线程唤醒它。这就是signal

Counting Semaphore (General semaphore)
* Not just 0 & 1
* Useful for representing resource

Producer/Consumer (bounded buffer)
* Data buffer[n]
* Semaphore spaces = n
* Semaphore filled = 0
* int prodId = 0;
* int conId = 0;

```java
produce:
    while (true) {
        Data d = produce(); // Get the data
        P(spaces); // P()
        buffer[prodId] = d;
        prodId = (prodId + 1) % n;
        V(filled);
    }

consumer:
    while (true) {
        p(filled);
        Data d = buffer[conId];
        conId = (conId + 1) % n;
        V(spaces);
        consume(d);
    }
```

Other Binary Semaphore
* Starts at 0 &rarr; signalling
    * Makes threads wait for each other

Drawbacks
* 2 ideas together - mutual exclusion & signalling(communication)
* P(), V() &rarr; separate, fragile design 
    * Java是用大括号括起来锁住的，就不会忘记

## Monitors
* ADT
    * shared data as private variable synchronized method, R/W can only happen through monitor 

* Dijkstra, Per Brinch Hansen
    * Package data and Operations are mutually-exclusive

* 兼具mutual exclusive && signalling 的性质 (可以上锁，也可以提供信号)
mutual exclusive part
```java
class Monitor {
    private int d1, d2, ...

    synchronized void foo() { ... }

    synchronized int bar() { ... }
}
```


signalling part
* Condition variable for thread communication, always associated with monitor (mutex)
* 2 ops
        | Pthread | Java |
        --- | ---
        `sleep()` | `wait()`
        `signal()` | `notify()` (can only invoke inside monitor)
* Behaviour
    * enter the monitor
        * check some state
        * Calling `wait()` inside a monitor will give it up & sleep (atomic)
        * When another thread calls `notify()`, sleeping thread may be woken up. Note that a thread that is woken cannot continue on until it has reacquired the lock

Q: How does it work?
A: We can think of that `2 queues` association with monitor
* Have a queue (set) of threads trying to get into the monitor (mq)
* Each CV implies another queue &rarr; cvq

atomic ops
* mq &rarr; all the threads trying to acquire a lock
* cq &rarr; all threads waiting on a conditional variable

```java
enter T:
    // while (anyone else in monitor) {
    //   add T to mq & put T to sleep
    // }

exit (T) {
    // take a thread out of mq & wake it up
}
    
wait (T cvq) {
    // add T to condition variable queue(cvq)
    // relax monitor & wake it up
}

notify() {
    // take a thread from cq & put to mq
}
   
notifyAll() {
    // move all threads from cvq to mq
}
    
```

* Notice that we wake one thread with `notify()`
* Upon being woken up, conditions may not hold; hence conditions should be checked again (typically with while loop)
* Spurious wakeups - may be woken up without being notified
    * Also solved by waiting in a while loop

## Different semantics for CV
* Signal & Continue
    * Notifier retains control of monitor
    * Woken thread competes with other threads for entry
* Signal & Wait
    * Notifier transfers monitor control to the woken thread
    * Notifier needs to re-acquire lock to continue
* Signal & Urgent Wait
    * Same as signal & wait, except a woken thread will give control back to the notifier once finished
* Reader/Writer Lock
    * DB context - only 1 write at a time
    * Multiple concurrent readers
