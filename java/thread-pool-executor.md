
# ThreadPoolExecutor Core Mechanism

## 1. The 7 Core Parameters

The behavior of a `ThreadPoolExecutor` is controlled by seven key parameters in its constructor:

```java
public ThreadPoolExecutor(
    int corePoolSize,           // 1. Baseline threads to keep alive
    int maximumPoolSize,        // 2. Max allowed threads
    long keepAliveTime,         // 3. Timeout for idle non-core threads
    TimeUnit unit,              // 4. Time unit for keepAliveTime
    BlockingQueue<Runnable> workQueue, // 5. Queue for holding tasks
    ThreadFactory threadFactory,// 6. Factory to create new threads (e.g., for naming)
    RejectedExecutionHandler handler // 7. Policy when saturated
)
````

## 2\. Task Execution Flow (Crucial)

This is a common misconception. The pool does **not** scale to `maximumPoolSize` immediately after `corePoolSize` is full. It fills the **Queue** first.

**The Order of Operations:**

1.  **Core:** If running threads \< `corePoolSize`, start a **new thread** immediately.
2.  **Queue:** If running threads \>= `corePoolSize`, try to put the task into the **`workQueue`**.
3.  **Max:** If the queue is **full** AND running threads \< `maximumPoolSize`, start a **new non-core thread**.
4.  **Reject:** If the queue is full AND running threads \>= `maximumPoolSize`, execute the **Rejection Policy**.

> **Mental Model:**
> Core full? -\> Queue.
> Queue full? -\> Expand to Max.
> Max full? -\> Reject.

## 3\. Rejection Policies

When the pool is completely saturated (Max threads reached + Queue full), one of the following handlers is triggered:

  * **`AbortPolicy` (Default):** Throws a `RejectedExecutionException`.
  * **`CallerRunsPolicy`:** The thread that submitted the task (usually the main thread) executes the task itself.
      * *Pro:* Provides a simple feedback mechanism (backpressure) that slows down the submission rate.
  * **`DiscardPolicy`:** Silently drops the task. No exception is thrown.
  * **`DiscardOldestPolicy`:** Drops the oldest task in the queue (the head) and retries submitting the current task.

## 4\. Why Avoid `Executors` Static Factories?

While `Executors.newFixedThreadPool()` is convenient, it is strictly forbidden in many guidelines (like the *Alibaba Java Development Manual*) for production use.

  * **`FixedThreadPool` / `SingleThreadPool`:**
      * They use a `LinkedBlockingQueue` with a capacity of `Integer.MAX_VALUE`.
      * **Risk:** A slow consumer can cause tasks to pile up indefinitely, leading to **OOM (Out Of Memory)**.
  * **`CachedThreadPool`:**
      * It sets `maximumPoolSize` to `Integer.MAX_VALUE`.
      * **Risk:** A burst of tasks can create thousands of threads, crashing the CPU or causing OOM.

**Best Practice:** Always explicitly define the parameters using the `ThreadPoolExecutor` constructor to limit both the queue size and the maximum thread count.

## 5\. Code Example (Best Practice)

```java
// Custom ThreadFactory (Recommended to use Guava or similar for meaningful names)
ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
        .setNameFormat("demo-pool-%d").build();

// Manual creation
ExecutorService pool = new ThreadPoolExecutor(
    5,      // corePoolSize
    10,     // maximumPoolSize
    60L,    // keepAliveTime
    TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100), // Bounded Queue (Capacity: 100)
    namedThreadFactory,
    new ThreadPoolExecutor.CallerRunsPolicy() // Policy: Caller runs
);

try {
    pool.execute(() -> {
        System.out.println("Task running in: " + Thread.currentThread().getName());
    });
} finally {
    pool.shutdown();
}

