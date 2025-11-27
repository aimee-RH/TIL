# Java Thread Safety & Locks: The Deep Dive

Thread safety means that a method or class instance can be used by multiple threads at the same time without any problems (race conditions, data corruption, or unexpected behavior).
To achieve this, we must control three critical aspects of the Java Memory Model (JMM): **Atomicity**, **Visibility**, and **Ordering**.

## 1. The Triad of Concurrency

### A. Atomicity 
**Definition:** An operation is atomic if it is indivisible. Either it happens completely, or it doesn't happen at all. No other thread can see an intermediate state.

* **The Problem:** `count++` is NOT atomic. It is actually three steps:
    1.  **Read** `count` from memory.
    2.  **Add** 1 to the value.
    3.  **Write** the new value back to memory.
    * *Race Condition:* If two threads do this simultaneously, they might overwrite each other's work.

* **The Fix:** Use `synchronized`, Locks, or Atomic classes.

### B. Visibility 
**Definition:** Changes made by one thread to shared data are immediately visible to other threads.

* **The Problem:** CPU Caches.
    * Thread A reads `flag = true` into its **CPU Cache**.
    * Thread B reads `flag = true` into its **CPU Cache**.
    * Thread A updates `flag = false` in its cache but hasn't flushed it to **Main Memory** yet.
    * Thread B continues to see `flag = true`.

* **The Fix:** Use `volatile` or `synchronized` (which forces a cache flush).

### C. Ordering 
**Definition:** The order in which instructions are executed.

* **The Problem:** To optimize performance, the Compiler and Processor may **reorder** instructions.
    * *Example:* `a = 1; b = 2;` might execute as `b = 2; a = 1;`.
    * In a single thread, this is fine (as-if-serial). In multi-threading, this breaks logic relying on dependency.

* **The Fix:** `volatile` (disables reordering via Memory Barriers) or `Happens-Before` rules.


## 2. Keyword: `volatile`

`volatile` is the lightweight synchronization mechanism.

### capabilities
1.  **Guarantees Visibility:** Reads/Writes go directly to Main Memory.
2.  **Guarantees Ordering:** Prevents instruction reordering.
3.  **❌ NO Atomicity:** It cannot protect `i++`.

### Use Case: The Status Flag
This is the most common correct use of `volatile`.

```java
public class Worker {
    // volatile guarantees that other threads see the change immediately
    private volatile boolean running = true; 

    public void work() {
        while (running) {
            // do task
        }
    }

    public void shutdown() {
        running = false; // Atomic write (boolean assignment is atomic)
    }
}
````

## 3\. Keyword: `synchronized` (Implicit Lock)

The most basic locking mechanism. It is distinct because it is managed by the **JVM**, not the code.

### How it works

It uses a **Monitor** lock (Monitor Object).

  * **Method Scope:** Locks the specific instance (`this`).
  * **Static Scope:** Locks the Class object (`Class<T>`), affecting all instances.
  * **Block Scope:** Locks a specific object provided.

### Code Examples

```java
public class Counter {
    private int count = 0;
    private final Object lockObj = new Object();

    // 1. Instance Method Lock (Locks 'this')
    public synchronized void increment() {
        count++;
    }

    // 2. Static Method Lock (Locks Counter.class)
    public static synchronized void globalAction() {
        // Global sync logic
    }

    // 3. Block Lock (Fine-grained control)
    // Recommended: Reduces the scope of the lock for better performance
    public void partialSync() {
        // ... code that doesn't need safety ...
        
        synchronized (lockObj) {
            count++;
        }
        
        // ... code that doesn't need safety ...
    }
}
```


## 4\. `ReentrantLock` (Explicit Lock)

Found in `java.util.concurrent.locks`. It provides more advanced features than `synchronized`.

### Comparison

| Feature | `synchronized` | `ReentrantLock` |
| :--- | :--- | :--- |
| **Release** | Automatic (by JVM) | **Manual** (Must use `finally`) |
| **Fairness** | Non-fair only | Choice (Fair or Non-fair) |
| **Interruptible** | No (Thread waits forever) | Yes (`lockInterruptibly`) |
| **Try Lock** | No | Yes (`tryLock` with timeout) |

### Standard Pattern (Try-Finally)

You **must** unlock in a `finally` block to prevent deadlocks if an exception occurs.

```java
import java.util.concurrent.locks.ReentrantLock;

public class ExplicitCounter {
    private final ReentrantLock lock = new ReentrantLock(); 
    // Pass 'true' to constructor for Fair Lock (FIFO)
    
    private int count = 0;

    public void increment() {
        lock.lock(); // Acquire lock
        try {
            count++;
        } finally {
            lock.unlock(); // CRITICAL: Always unlock here
        }
    }
    
    // Example: Try Lock (Avoid waiting forever)
    public void safeAction() {
        if (lock.tryLock()) {
            try {
                // do work
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println("Could not get lock, doing something else...");
        }
    }
}
```


## 5\. CAS & Atomic Variables (Optimistic Locking)

**CAS (Compare-And-Swap)** is the hardware primitive that enables concurrency without heavy locks.

### Logic

*"I think the value at address V is A. If it is A, update it to B. If it's not A (someone changed it), tell me, and I will try again."*

### Atomic Classes

Java uses CAS in the `java.util.concurrent.atomic` package.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    // Thread-safe without 'synchronized'
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        // Internally uses Unsafe.compareAndSwapInt loop
        count.incrementAndGet(); 
    }
    
    public int get() {
        return count.get();
    }
}
```

### The ABA Problem

A classic CAS issue.

1.  Thread 1 reads value **A**.
2.  Thread 2 changes **A -\> B**, then changes **B -\> A**.
3.  Thread 1 checks value, sees **A**, and thinks nothing changed.
      * **Fix:** Use `AtomicStampedReference`, which adds a version number (A1 -\> B2 -\> A3).


## 6\. Deadlock (The Danger Zone)

Deadlock happens when two threads wait for each other to release resources specifically in a cycle.

**Scenario:**

  * Thread 1 holds Lock A, wants Lock B.
  * Thread 2 holds Lock B, wants Lock A.
  * Result: They wait forever.

**Prevention:**

1.  **Fixed Ordering:** Always acquire locks in the same order (Always A then B).
2.  **Timeout:** Use `ReentrantLock.tryLock(time)` to give up if waiting too long.

