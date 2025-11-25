# Java 21 Virtual Threads (Project Loom)

## 1. The Paradigm Shift

Before Java 21, threads (Platform Threads) were expensive wrappers around OS threads. We used **Thread Pools** to limit the cost of creating them.

**With Virtual Threads:**
* Threads are cheap user-mode entities managed by the JVM.
* Blocking operations (I/O) unmount the virtual thread, freeing the underlying carrier thread.
* **Cost:** Creating a virtual thread is almost as cheap as creating a String.

## 2. No More Pooling

**Anti-Pattern:** Do **NOT** pool virtual threads.

Since they are cheap and disposable, you should create a new virtual thread for every single task.

* **Old Way (Platform Threads):** "I have 1000 tasks, I'll use a pool of 50 threads."
* **New Way (Virtual Threads):** "I have 1000 tasks, I'll create 1000 virtual threads."

## 3. How to Use

### Using Executors (The New Standard)
Java 21 introduced a new executor specifically for this. It does not limit the number of threads; it starts a new virtual thread for each submitted task.

```java
// 1. New factory method
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1)); // Blocking is cheap now!
            return i;
        });
    });
} // Executor auto-closes and waits for all tasks to finish

// Start directly
Thread.startVirtualThread(() -> {
    System.out.println("Running in virtual thread");
});

// Using Builder
Thread vThread = Thread.ofVirtual()
    .name("my-vthread")
    .start(() -> { 
        // task 
    });
